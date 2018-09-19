# Configure OpenFaaS access policies

Kubernetes namespaces alone offer only a logical separation between workloads.
To prohibit functions from calling each other or from reaching 
the OpenFaaS core services we need to create Istio Mixer rules.

Deny access to OpenFaaS core services from the `openfaas-fn` namespace except for system functions:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: denier
metadata:
  name: denyhandler
  namespace: openfaas
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: config.istio.io/v1alpha2
kind: checknothing
metadata:
  name: denyrequest
  namespace: openfaas
spec:
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: denyopenfaasfn
  namespace: openfaas
spec:
  match: destination.namespace == "openfaas" && source.namespace == "openfaas-fn" && source.labels["role"] != "openfaas-system"
  actions:
  - handler: denyhandler.denier
    instances: [ denyrequest.checknothing ]
```

Save the above resources as of-rules.yaml and then apply it:

```bash
kubectl apply -f ./of-rules.yaml
```

Deny access to functions except for OpenFaaS core services:

```yaml
apiVersion: config.istio.io/v1alpha2
kind: denier
metadata:
  name: denyhandler
  namespace: openfaas-fn
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: config.istio.io/v1alpha2
kind: checknothing
metadata:
  name: denyrequest
  namespace: openfaas-fn
spec:
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: denyopenfaasfn
  namespace: openfaas-fn
spec:
  match: destination.namespace == "openfaas-fn" && source.namespace != "openfaas" && source.labels["role"] != "openfaas-system"
  actions:
  - handler: denyhandler.denier
    instances: [ denyrequest.checknothing ]
```

Save the above resources as of-functions-rules.yaml and then apply it:

```bash
kubectl apply -f ./of-functions-rules.yaml
```

Next: [Install OpenFaaS with Helm](03-openfaas-setup.md)
