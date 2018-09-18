# Configure OpenFaaS Gateway to receive external traffic

Create an Istio virtual service for OpenFaaS Gateway (replace `example.com` with your domain):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gateway
  namespace: openfaas
spec:
  hosts:
  - "openfaas.example.com"
  gateways:
  - public-gateway.istio-system.svc.cluster.local
  http:
  - route:
    - destination:
        host: gateway
    timeout: 30s
```

Save the above resource as of-virtual-service.yaml and then apply it:

```bash
kubectl apply -f ./of-virtual-service.yaml
```

Wait for OpenFaaS Gateway to come online:

```bash
watch curl -v https://openfaas.example.com/heathz 
```

Save your credentials in faas-cli store:

```bash
echo $password | faas-cli login -g https://openfaas.example.com -u admin --password-stdin
```

Next: [Canary deployments for OpenFaaS functions](05-canary.md)
