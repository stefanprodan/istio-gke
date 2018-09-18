# Configure Istio Gateway with Let's Encrypt wildcard certificate

![istio-letsencrypt](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/istio-cert-manager-gcp.png)

Create a Istio Gateway in istio-system namespace with HTTPS redirect:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*"
    tls:
      mode: SIMPLE
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
```

Save the above resource as istio-gateway.yaml and then apply it:

```bash
kubectl apply -f ./istio-gateway.yaml
```

Create a service account with Cloud DNS admin role (replace `my-gcp-project` with your project ID):

```bash
GCP_PROJECT=my-gcp-project

gcloud iam service-accounts create dns-admin \
--display-name=dns-admin \
--project=${GCP_PROJECT}

gcloud iam service-accounts keys create ./gcp-dns-admin.json \
--iam-account=dns-admin@${GCP_PROJECT}.iam.gserviceaccount.com \
--project=${GCP_PROJECT}

gcloud projects add-iam-policy-binding ${GCP_PROJECT} \
--member=serviceAccount:dns-admin@${GCP_PROJECT}.iam.gserviceaccount.com \
--role=roles/dns.admin
```

Create a Kubernetes secret with the GCP Cloud DNS admin key:

```bash
kubectl create secret generic cert-manager-credentials \
--from-file=./gcp-dns-admin.json \
--namespace=istio-system
```

Create a letsencrypt issuer for CloudDNS (replace `email@example.com` with a valid email address and `my-gcp-project` with your project ID):

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: istio-system
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    dns01:
      providers:
      - name: cloud-dns
        clouddns:
          serviceAccountSecretRef:
            name: cert-manager-credentials
            key: gcp-dns-admin.json
          project: my-gcp-project
```

Save the above resource as letsencrypt-issuer.yaml and then apply it:

```bash
kubectl apply -f ./letsencrypt-issuer.yaml
```

Create a wildcard certificate (replace `example.com` with your domain):

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: istio-gateway
  namespace: istio-system
spec:
  secretname: istio-ingressgateway-certs
  issuerRef:
    name: letsencrypt-prod
  commonName: "*.example.com"
  dnsNames:
  - istio.example.com
  acme:
    config:
    - dns01:
        provider: cloud-dns
      domains:
      - "*.example.com"
      - "example.com"
```

Save the above resource as of-cert.yaml and then apply it:

```bash
kubectl apply -f ./of-cert.yaml
```

In a couple of seconds cert-manager should fetch a wildcard certificate from letsencrypt.org:

```bash
kubectl -n istio-system logs deployment/certmanager -f
Certificate issued successfully
```

Recreate Istio ingress gateway pods:

```bash
kubectl -n istio-system delete pods -l istio=ingressgateway
```

Note that Istio gateway doesn't reload the certificates from the TLS secret on cert-manager renewal. 
Since the GKE cluster is made out of preemptible VMs the gateway pods will be replaced once every 24h, if your not using 
preemptible nodes then you need to manually kill the gateway pods every two months before the certificate expires.
