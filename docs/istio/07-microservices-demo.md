# Microservices demo: A/B testing and canary deployments

To experiment with different traffic routing techniques 
I've created a Helm chart for [podinfo](https://github.com/stefanprodan/k8s-podinfo) that lets you chain multiple 
services and wraps all the Istio objects needs for A/B testing and canary deployments.

Using the podinfo chart you will be installing three microservices: frontend, backend and data store. 
Each of these services can have two versions running in parallel, the versions are called blue and green.
The assumption is that for the frontend you'll be running A/B testing based on the user agent HTTP header. 
The green frontend is not backwards compatible with the blue backend so you'll route all requests from the green 
frontend to the green backend. For the data store you'll be running performance testing. Both backend versions are 
compatible with the blue and green data store so you'll be splitting the traffic between blue and green data stores 
and compare the requests latency and error rate to determine if the green store performs 
better than the blue one.

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

Enable mutual TLS for the `demo` namespace:

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: default
  namespace: demo
spec:
  peers:
  - mtls:
      mode: PERMISSIVE
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

### Initial state

![initial-state](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/routing-initial-state.png)

Create a frontend release exposed outside the service mesh on the podinfo sub-domain (replace `example.com` with your domain):

```yaml
host: podinfo.example.com
exposeHost: true

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  message: "Greetings from the blue frontend"
  backend: http://backend:9898/api/echo

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as frontend.yaml and then install it:

```bash
helm install --name frontend sp/podinfo-istio \
--namespace demo \
-f ./frontend.yaml
```

Create a backend release:

```yaml
host: backend

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  backend: http://store:9898/api/echo

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as backend.yaml and then install it:

```bash
helm install --name backend sp/podinfo-istio \
--namespace demo \
-f ./backend.yaml
```

Create a store release:

```yaml
host: store

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  weight: 100

green:
  # disabled (all traffic goes to blue)
  replicas: 0
```

Save the above resource as store.yaml and then install it:

```bash
helm install --name store sp/podinfo-istio \
--namespace demo \
-f ./store.yaml
```

Open `https://podinfo.exmaple.com` in your browser, you should see a greetings message from the blue version.
Clicking on the ping button will make a call that spans across all microservices.

Access Jaeger dashboard using port forwarding:

```bash
kubectl -n istio-system port-forward deployment/istio-tracing 16686:16686 
```

Navigate to `http://localhost:16686` and select `store` from the service dropdown. You should see a trace for each ping.

![jaeger-trace](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/jaeger-trace-list.png)

Istio tracing is able to capture the ping call spanning across all microservices because podinfo forwards the Zipkin HTTP 
headers. When a HTTP request reaches the Istio Gateway, Envoy will inject a series of headers used for tracing. When podinfo 
calls a backend service, will copy the headers from the incoming HTTP request:

```go
func copyTracingHeaders(from *http.Request, to *http.Request) {
	headers := []string{
		"x-request-id",
		"x-b3-traceid",
		"x-b3-spanid",
		"x-b3-parentspanid",
		"x-b3-sampled",
		"x-b3-flags",
		"x-ot-span-context",
	}

	for i := range headers {
		headerValue := from.Header.Get(headers[i])
		if len(headerValue) > 0 {
			to.Header.Set(headers[i], headerValue)
		}
	}
}
```

### Desired state

![desired-state](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/routing-desired-state.png)

Change the frontend definition to route traffic coming from Safari users to the green deployment:

```yaml
host: podinfo.example.com
exposeHost: true

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  message: "Greetings from the blue frontend"
  backend: http://backend:9898/api/echo

green:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
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
            regex: "^(v{0,1})1\\.2\\.([0-9]).*"
  message: "Greetings from the green frontend"
  backend: http://backend:9898/api/echo
```

Save the above resource and apply it:

```bash
helm upgrade --install frontend sp/podinfo-istio \
--namespace demo \
-f ./frontend.yaml
```

Change the backend definition to receive traffic based on source labels. The blue frontend will be routed to the blue
backend and the green frontend to the green backend:

```yaml
# expose the backend deployment inside the cluster on backend.demo
host: backend

blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  backend: http://store:9898/api/echo

green:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.2.0
  routing:
    # target green callers
    - match:
      - sourceLabels:
          color: green
  backend: http://store:9898/api/echo
```

Save the above resource and apply it:

```bash
helm upgrade --install backend sp/podinfo-istio \
--namespace demo \
-f ./backend.yaml
```

Change the store definition to route 80% of the traffic to the blue deployment and 20% to the green one:

```yaml
# expose the store deployment inside the cluster on store.demo
host: store

# load balance 80/20 between blue and green
blue:
  replicas: 2
  image: quay.io/stefanprodan/podinfo:1.1.1
  weight: 80

green:
  replicas: 1
  image: quay.io/stefanprodan/podinfo:1.2.0
```

Save the above resource and apply it:

```bash
helm upgrade --install store sp/podinfo-istio \
--namespace demo \
-f ./store.yaml
```
