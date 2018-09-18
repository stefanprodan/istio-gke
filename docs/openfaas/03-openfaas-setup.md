# Install OpenFaaS with Helm

Before installing OpenFaaS you need to provide the basic authentication credential for the OpenFaaS gateway.

Create a secret named `basic-auth` in the `openfaas` namespace:

```bash
# generate a random password
password=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

kubectl -n openfaas create secret generic basic-auth \
--from-literal=basic-auth-user=admin \
--from-literal=basic-auth-password=$password 
```

Add the OpenFaaS `helm` repository:

```bash
$ helm repo add openfaas https://openfaas.github.io/faas-netes/
```

Install OpenFaaS with Helm:

```bash
helm upgrade --install openfaas ./chart/openfaas \
--namespace openfaas \
--set functionNamespace=openfaas-fn \
--set operator.create=true \
--set securityContext=true \
--set basic_auth=true \
--set exposeServices=false \
--set operator.createCRD=true
```

Verify that OpenFaaS workloads are running:

```bash
kubectl -n openfaas get pods
```

Next: [Configure OpenFaaS Gateway to receive external traffic](04-gateway-config.md)
