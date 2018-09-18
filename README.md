# Istio service mesh walkthrough 

This guide walks you through setting up Istio on Google Kubernetes Engine.

### Labs

* [Prerequisites - client tools](/docs/istio/01-prerequisites.md)
* [GKE cluster setup](/docs/istio/02-gke-setup.md)
* [Cloud DNS setup](/docs/istio/03-clouddns-setup.md)
* [Install Istio with Helm](/docs/istio/04-istio-setup.md)
* [Configure Istio Gateway with Let's Encrypt wildcard certificate](/docs/istio/05-letsencrypt-setup.md)
* [Expose services outside the service mesh](/docs/istio/06-grafana-config.md)

## OpenFaaS

This guide walks you through setting up OpenFaaS with Istio on Google Kubernetes Engine.

![openfaas-istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-diagram.png)

At the end of this guide you will be running OpenFaaS with the following characteristics:

* secure OpenFaaS ingress with Let’s Encrypt TLS and authentication
* encrypted communication between OpenFaaS core services and functions with Istio mutual TLS
* isolated functions with Istio Mixer rules
* Jaeger tracing and Prometheus monitoring for function calls
* canary deployments for OpenFaaS functions 

### Labs

[Configure OpenFaaS mutual TLS](/docs/openfaas/01-mtls-config.md)
[Configure OpenFaaS access policies](/docs/openfaas/02-mixer-rules.md)
[Install OpenFaaS with Helm](/docs/openfaas/03-openfaas-setup.md)
[Configure OpenFaaS Gateway to receive external traffic](/docs/openfaas/04-gateway-config.md)
[Canary deployments for OpenFaaS functions](/docs/openfaas/05-canary.md)
