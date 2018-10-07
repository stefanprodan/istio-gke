# Expose Grafana outside the service mesh

In order to expose services via the Istio Gateway you have to create a Virtual Service attached to Istio Gateway.

Create a virtual service in `istio-system` namespace for Grafana (replace `example.com` with your domain):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana
  namespace: istio-system
spec:
  hosts:
  - "grafana.example.com"
  gateways:
  - public-gateway.istio-system.svc.cluster.local
  http:
  - route:
    - destination:
        host: grafana
    timeout: 30s
```

Save the above resource as grafana-virtual-service.yaml and then apply it:

```bash
kubectl apply -f ./grafana-virtual-service.yaml
```

Navigate to `http://grafana.example.com` in your browser and you should be redirected to the HTTPS version.

Check that HTTP2 is enabled:

```bash
curl -I --http2 https://grafana.example.com

HTTP/2 200 
content-type: text/html; charset=UTF-8
x-envoy-upstream-service-time: 3
server: envoy
```

Next: [A/B testing and canary deployments demo](/docs/apps/00-index.md)
