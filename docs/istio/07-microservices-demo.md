# Microservices demo: A/B testing and canary deployments

Add podinfo Helm repository:

```bash
helm repo add sp https://stefanprodan.github.io/k8s-podinfo
```

Create a namespace with Istio sidecar injection enabled:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    istio-injection: enabled
  name: demo
```

Save the above resource as demo.yaml and then apply it:

```bash
kubectl apply -f ./demo.yaml
```

Enable mTLS for the `demo` namespace:

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: default
  namespace: demo
spec:
  peers:
  - mtls: {}
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: default
  namespace: demo
spec:
  host: "*.demo.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Save the above resource as demo-mtls.yaml and then apply it:

```bash
kubectl apply -f ./demo-mtls.yaml
```

### A/B Testing initial state

![initial-state](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/routing-initial-state.png)

Frontend:

```yaml
host: podinfo.example.com
exposeHost: true

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  message: "Greetings from the blue frontend"
  backend: http://backend:9898/api/echo

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as frontend.yaml and then install it:

```bash
helm upgrade --install frontend sp/podinfo-istio \
--namespace demo \
- f ./frontend.yaml
```

Backend:

```yaml
host: backend

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  backend: http://store:9898/api/echo

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as backend.yaml and then install it:

```bash
helm upgrade --install backend sp/podinfo-istio \
--namespace demo \
- f ./backend.yaml
```

Data store:

```yaml
host: store

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  weight: 100

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as data.yaml and then install it:

```bash
helm upgrade --install store sp/podinfo-istio \
--namespace demo \
- f ./data.yaml
```

### A/B Testing desired state

![desired-state](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/routing-desired-state.png)

Frontend:

```yaml
# expose the frontend deployment outside the cluster
host: podinfo.example.com
exposeHost: true

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  message: "Greetings from the blue frontend"
  backend: http://backend:9898/api/echo

green:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.1
  routing:
    # target Safari
    - match:
      - headers:
          user-agent:
            regex: "^(?!.*Chrome).*Safari.*"
    # target API clients by version
    - match:
      - headers:
          x-api-version:
            regex: "^(v{0,1})1\\.2\\.([1-9]).*"
  message: "Greetings from the green frontend"
  backend: http://backend:9898/api/echo
```

Backend:

```yaml
# expose the backend deployment inside the cluster on backend.demo
host: backend

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  backend: http://store:9898/api/echo

green:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.1
  routing:
    # target green callers
    - match:
      - sourceLabels:
          color: green
  backend: http://store:9898/api/echo
```

Data store:

```yaml
# expose the store deployment inside the cluster on store.demo
host: store

# load balance 80/20 between blue and green
blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  weight: 80

green:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.1
```
