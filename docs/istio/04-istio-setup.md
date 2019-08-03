# Install Istio with Helm

Add Istio Helm repository:

```bash
export ISTIO_VER="1.2.3"

helm repo add istio.io https://storage.googleapis.com/istio-release/releases/${ISTIO_VER}/charts
```

Installing the Istio custom resource definitions:

```bash
helm upgrade -i istio-init istio.io/istio-init --wait --namespace istio-system
```

Wait for Istio CRDs to be deployed:

```bash
kubectl -n istio-system wait --for=condition=complete job/istio-init-crd-10
kubectl -n istio-system wait --for=condition=complete job/istio-init-crd-11
kubectl -n istio-system wait --for=condition=complete job/istio-init-crd-12
```

Create a secret for Grafana credentials:

```bash
# generate a random password
PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

kubectl -n istio-system create secret generic grafana \
--from-literal=username=admin \
--from-literal=passphrase="$PASSWORD"
```

Configure Istio with Prometheus, Jaeger, and cert-manager and set your load balancer IP:

```yaml
# ingress configuration
gateways:
  enabled: true
  istio-ingressgateway:
    type: LoadBalancer
    loadBalancerIP: "35.198.98.90"
    autoscaleEnabled: true
    autoscaleMax: 2
    
# common settings
global:
  # sidecar settings
  proxy:
    resources:
      requests:
        cpu: 10m
        memory: 64Mi
      limits:
        cpu: 2000m
        memory: 256Mi
  controlPlaneSecurityEnabled: false
  mtls:
    enabled: false
  useMCP: true

# pilot configuration
pilot:
  enabled: true
  autoscaleEnabled: true
  sidecar: true
  resources:
    requests:
      cpu: 10m
      memory: 128Mi

# sidecar-injector webhook configuration
sidecarInjectorWebhook:
  enabled: true

# security configuration
security:
  enabled: true

# galley configuration
galley:
  enabled: true

# mixer configuration
mixer:
  policy:
    enabled: false
    replicaCount: 1
    autoscaleEnabled: true
  telemetry:
    enabled: true
    replicaCount: 1
    autoscaleEnabled: true
  resources:
    requests:
      cpu: 10m
      memory: 128Mi

# addon prometheus configuration
prometheus:
  enabled: true
  scrapeInterval: 5s

# addon jaeger tracing configuration
tracing:
  enabled: true

# addon grafana configuration
grafana:
  enabled: true
  security:
    enabled: true
```

Save the above file as `my-istio.yaml` and install Istio with Helm:

```bash
helm upgrade --install istio istio.io/istio \
--namespace=istio-system \
-f ./my-istio.yaml
```

Verify that Istio workloads are running:

```bash
watch kubectl -n istio-system get pods
```

Next: [Configure Istio Gateway with Let's Encrypt wildcard certificate](05-letsencrypt-setup.md)
