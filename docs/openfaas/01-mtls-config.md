# Configure OpenFaaS mutual TLS

An OpenFaaS instance is composed out of two namespaces: one for the core services and one for functions. 
In order to secure the communication between core services and functions we need to enable mutual TLS on both namespaces.

Create the OpenFaaS namespaces with Istio sidecar injection enabled:

```bash
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

Enable mTLS on `openfaas` namespace:

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: default
  namespace: openfaas
spec:
  peers:
  - mtls: {}
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: default
  namespace: openfaas
spec:
  host: "*.openfaas.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Save the above resource as of-mtls.yaml and then apply it:

```bash
kubectl apply -f ./of-mtls.yaml
```

Allow plaintext traffic to NATS:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
    name: "nats-no-mtls"
    namespace: {{ .Release.Namespace }}
spec:
    host: "nats.openfaas.svc.cluster.local"
    trafficPolicy:
        tls:
            mode: DISABLE
```

Save the above resource as of-nats-no-mtls.yaml and then apply it:

```bash
kubectl apply -f ./of-nats-no-mtls.yaml
```

Enable mTLS on `openfaas-fn` namespace:

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: default
  namespace: openfaas-fn
spec:
  peers:
  - mtls: {}
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: default
  namespace: openfaas-fn
spec:
  host: "*.openfaas-fn.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Save the above resource as of-functions-mtls.yaml and then apply it:

```bash
kubectl apply -f ./of-functions-mtls.yaml
```

Next: [Configure OpenFaaS access policies](02-mixer-rules.md)
