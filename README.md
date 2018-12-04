# Istio service mesh guides 

![istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/istio-gcp-overview.png)

[Istio GKE setup](/docs/istio/00-index.md)

* [Prerequisites - client tools](/docs/istio/01-prerequisites.md)
* [GKE cluster setup](/docs/istio/02-gke-setup.md)
* [Cloud DNS setup](/docs/istio/03-clouddns-setup.md)
* [Install Istio with Helm](/docs/istio/04-istio-setup.md)
* [Configure Istio Gateway with Let's Encrypt wildcard certificate](/docs/istio/05-letsencrypt-setup.md)
* [Expose services outside the service mesh](/docs/istio/06-grafana-config.md)

[Progressive delivery walkthrough](docs/apps/00-index.md)

* [Automated canary deployments with Flagger](/docs/apps/01-canary-flagger.md)
* [A/B testing for a micro-service stack with Helm](/docs/apps/02-ab-testing-helm.md)

[OpenFaaS service mesh walkthrough](docs/openfaas/00-index.md)

* [Configure OpenFaaS mutual TLS](/docs/openfaas/01-mtls-config.md)
* [Configure OpenFaaS access policies](/docs/openfaas/02-mixer-rules.md)
* [Install OpenFaaS with Helm](/docs/openfaas/03-openfaas-setup.md)
* [Configure OpenFaaS Gateway to receive external traffic](/docs/openfaas/04-gateway-config.md)
* [Canary deployments for OpenFaaS functions](/docs/openfaas/05-canary.md)
