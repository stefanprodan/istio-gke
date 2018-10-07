# Istio GKE setup

This guide walks you through setting up Istio on Google Kubernetes Engine.

![istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/istio-gcp-overview.png)

At the end of this guide you will be running Istio with the following characteristics:

* secure Istio ingress gateway with Let’s Encrypt TLS
* encrypted communication between Kubernetes workloads with Istio mutual TLS
* Jaeger tracing 
* Prometheus and Grafana monitoring
* canary deployments, A/B testing and traffic mirroring capabilities

### Labs

* [Prerequisites - client tools](01-prerequisites.md)
* [GKE cluster setup](02-gke-setup.md)
* [Cloud DNS setup](03-clouddns-setup.md)
* [Install Istio with Helm](04-istio-setup.md)
* [Configure Istio Gateway with Let's Encrypt wildcard certificate](05-letsencrypt-setup.md)
* [Expose services outside the service mesh](06-grafana-config.md)
