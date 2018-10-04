# Canary deployments for OpenFaaS functions

![openfaas-canary](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-canary.png)

Create a general available release for the `env` function version 1.0.0:

```yaml
apiVersion: openfaas.com/v1alpha2
kind: Function
metadata:
  name: env
  namespace: openfaas-fn
spec:
  name: env
  image: stefanprodan/of-env:1.0.0
  resources:
    requests:
      memory: "32Mi"
      cpu: "10m"
  limits:
    memory: "64Mi"
    cpu: "100m"
```

Save the above resources as env-ga.yaml and then apply it:

```bash
kubectl apply -f ./env-ga.yaml
```

Create a canary release for version 1.1.0:

```yaml
apiVersion: openfaas.com/v1alpha2
kind: Function
metadata:
  name: env-canary
  namespace: openfaas-fn
spec:
  name: env-canary
  image: stefanprodan/of-env:1.1.0
  resources:
    requests:
      memory: "32Mi"
      cpu: "10m"
  limits:
    memory: "64Mi"
    cpu: "100m"
```

Save the above resources as env-canary.yaml and then apply it:

```bash
kubectl apply -f ./env-canary.yaml
```

Create an Istio virtual service with 10% traffic going to canary:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: env
  namespace: openfaas-fn
spec:
  hosts:
  - env
  http:
  - route:
    - destination:
        host: env
      weight: 90
    - destination:
        host: env-canary
      weight: 10
    timeout: 30s
```

Save the above resources as env-virtual-service.yaml and then apply it:

```bash
kubectl apply -f ./env-virtual-service.yaml
```

Test traffic routing (one in ten calls should hit the canary release):

```bash
 while true; do sleep 1; curl -sS https://openfaas.example.com/function/env | grep HOSTNAME; done

HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-59bf48fb9d-cjsjw
HOSTNAME=env-canary-5dffdf4458-4vnn2
```

Access Jaeger dashboard using port forwarding:

```bash
kubectl -n istio-system port-forward deployment/istio-tracing 16686:16686
```

Tracing the general available release:

![ga-trace](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-ga-trace.png)

Tracing the canary release:

![canary-trace](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-canary-trace.png)

Monitor ga vs canary success rate and latency with Grafana:

![canary-prom](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-canary-prom.png)
