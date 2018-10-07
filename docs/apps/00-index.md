# Progressive delivery walkthrough

This guide shows you how to route traffic between different versions of a service and how to automate canary deployments.

![flagger-overview](https://raw.githubusercontent.com/stefanprodan/flagger/master/docs/diagrams/flagger-overview.png)

At the end of this guide you will be deploying a series of micro-services with the following characteristics:

* A/B testing for frontend services
* Source/Destination based routing for backend services
* Progressive deployments gated by Prometheus 

### Labs

* [Automated canary deployments with Flagger](01-canary-flagger.md)
* [A/B testing for a micro-service stack with Helm](02-ab-testing-helm.md)
