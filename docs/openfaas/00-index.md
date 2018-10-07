# OpenFaaS service mesh walkthrough

This guide walks you through setting up OpenFaaS with Istio on Google Kubernetes Engine.

![openfaas-istio](https://github.com/stefanprodan/istio-gke/blob/master/docs/screens/openfaas-istio-diagram.png)

At the end of this guide you will be running OpenFaaS with the following characteristics:

* secure OpenFaaS ingress with Let’s Encrypt TLS and authentication
* encrypted communication between OpenFaaS core services and functions with Istio mutual TLS
* isolated functions with Istio Mixer rules
* Jaeger tracing and Prometheus monitoring for function calls
* canary deployments for OpenFaaS functions 

### Labs

* [Configure OpenFaaS mutual TLS](01-mtls-config.md)
* [Configure OpenFaaS access policies](02-mixer-rules.md)
* [Install OpenFaaS with Helm](03-openfaas-setup.md)
* [Configure OpenFaaS Gateway to receive external traffic](04-gateway-config.md)
* [Canary deployments for OpenFaaS functions](05-canary.md)
