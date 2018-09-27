# Istio service mesh guides 

## Istio GKE setup

This guide walks you through setting up Istio on Google Kubernetes Engine.

![istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/istio-gcp-overview.png)

At the end of this guide you will be running Istio with the following characteristics:

* secure Istio ingress gateway with Let’s Encrypt TLS
* encrypted communication between Kubernetes workloads with Istio mutual TLS
* Jaeger tracing 
* Prometheus and Grafana monitoring
* canary deployments, A/B testing and traffic mirroring capabilities

### Labs

* [Prerequisites - client tools](/docs/istio/01-prerequisites.md)
* [GKE cluster setup](/docs/istio/02-gke-setup.md)
* [Cloud DNS setup](/docs/istio/03-clouddns-setup.md)
* [Install Istio with Helm](/docs/istio/04-istio-setup.md)
* [Configure Istio Gateway with Let's Encrypt wildcard certificate](/docs/istio/05-letsencrypt-setup.md)
* [Expose services outside the service mesh](/docs/istio/06-grafana-config.md)

## Progressive delivery walkthrough

This guide shows you how to route traffic between different versions of a service and how to automate canary deployments.

![steerer-overview](https://raw.githubusercontent.com/stefanprodan/steerer/master/docs/diagrams/steerer-overview.png)

At the end of this guide you will be deploying a series of micro-services with the following characteristics:

* A/B testing for frontend services
* Source/Destination based routing for backend services
* Progressive deployments gated by Prometheus 

### Labs

* [A/B testing for a micro-service stack with Helm](/docs/apps/01-ab-testing-helm.md)
* [Automated canary deployments with Steerer](/docs/apps/02-canary-steerer.md)

## OpenFaaS service mesh walkthrough

This guide walks you through setting up OpenFaaS with Istio on Google Kubernetes Engine.

![openfaas-istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-diagram.png)

At the end of this guide you will be running OpenFaaS with the following characteristics:

* secure OpenFaaS ingress with Let’s Encrypt TLS and authentication
* encrypted communication between OpenFaaS core services and functions with Istio mutual TLS
* isolated functions with Istio Mixer rules
* Jaeger tracing and Prometheus monitoring for function calls
* canary deployments for OpenFaaS functions 

### Labs

* [Configure OpenFaaS mutual TLS](/docs/openfaas/01-mtls-config.md)
* [Configure OpenFaaS access policies](/docs/openfaas/02-mixer-rules.md)
* [Install OpenFaaS with Helm](/docs/openfaas/03-openfaas-setup.md)
* [Configure OpenFaaS Gateway to receive external traffic](/docs/openfaas/04-gateway-config.md)
* [Canary deployments for OpenFaaS functions](/docs/openfaas/05-canary.md)
