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

### Restrict access with Mixer rules

Let's assume the frontend service has a vulnerability and a bad actor can execute arbitrary commands in the frontend container.
If someone gains access to the frontend service, from there he/she can issue API calls to the backend and data store service.

In order to simulate this you can exec into the frontend container and curl the data store API:

```bash
kubectl -n demo exec -it frontend-blue-675b4dff4b-xhg9d -c podinfod sh

~ $ curl -v http://store:9898
* Connected to store (10.31.250.154) port 9898 (#0)
```

There is no reason why the frontend service should have access to the data store, only the backend service should be 
able to issue API calls to the store service. With Istio you can define access rules and restrict access based on source and 
destination.

Let's create an Istio config that denies access to the data store unless the caller is the backend service:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: denier
metadata:
  name: denyhandler
  namespace: demo
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: config.istio.io/v1alpha2
kind: checknothing
metadata:
  name: denyrequest
  namespace: demo
spec:
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: denystore
  namespace: demo
spec:
  match:  destination.labels["app"] == "store" && source.labels["app"] != "backend"
  actions:
  - handler: denyhandler.denier
    instances: [ denyrequest.checknothing ]
```

Save the above resource as demo-rules.yaml and then apply it:

```bash
kubectl apply -f ./demo-rules.yaml
```

Now if try to call the data store from the frontend container Istio Mixer will deny access:

```bash
kubectl -n demo exec -it frontend-blue-675b4dff4b-xhg9d -c podinfod sh

~ $ watch curl -s http://store:9898
PERMISSION_DENIED:denyhandler.denier.demo:Not allowed
```

The permission denied error can be observed in Grafana. Open the Istio Workload dashboard, select the demo namespace and
podinfo-blue workload from the dropdown, scroll to outbound services and you'll see the HTTP 403 errors:

![grafana-403](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/grafana-403-errors.png)
