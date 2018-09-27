# Automated canary deployments with Istio and Prometheus

[Steerer](https://github.com/stefanprodan/steerer) is a Kubernetes operator that automates the promotion of
canary deployments using Istio routing for traffic shifting and Prometheus metrics for canary analysis.

### Install 

Deploy Steerer in the `istio-system` namespace using Helm:

```bash
# add Steerer Helm repo
helm repo add steerer https://stefanprodan.github.io/steerer

# install or upgrade Steerer
helm upgrade --install steerer steerer/steerer \
--namespace=istio-system \
--set metricsServer=http://prometheus.istio-system:9090 \
--set controlLoopInterval=1m
```

### Usage

Steerer requires two Kubernetes deployments: one for the version you want to upgrade called _primary_ and one for the _canary_.
Each deployment must have a corresponding ClusterIP service that exposes a port named http or https.
These services are used as destinations in a Istio virtual service.

Gated rollout stages:

* scan for deployments marked for rollout 
* check Istio virtual service routes are mapped to primary and canary ClusterIP services
* check primary and canary deployments status
    * halt rollout if a rolling update is underway
    * halt rollout if pods are unhealthy
* increase canary traffic weight percentage from 0% to 10%
* check canary HTTP success rate
    * halt rollout if percentage is under the specified threshold
* increase canary traffic wight by 10% till it reaches 100% 
    * halt rollout while canary success rate is under the threshold
    * halt rollout if the primary or canary deployment becomes unhealthy 
    * halt rollout while canary deployment is being scaled up/down by HPA
* promote canary to primary
    * copy canary deployment spec template over primary
* wait for primary rolling update to finish
    * halt rollout if pods are unhealthy
* route all traffic to primary
* scale to zero the canary deployment
* mark rollout as finished
* wait for the canary deployment to be updated (revision bump) and start over

### Example

Create a test namespace with Istio sidecard injection enabled:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
  labels:
    istio-injection: enabled
```

Create the primary deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: test
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfod
        image: quay.io/stefanprodan/podinfo:1.1.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9898
          name: http
        command:
        - ./podinfo
        - --port=9898
        - --level=info
        env:
        - name: PODINFO_UI_COLOR
          value: blue
        livenessProbe:
          exec:
            command:
            - podcli
            - check
            - http
            - localhost:9898/healthz
        readinessProbe:
          exec:
            command:
            - podcli
            - check
            - http
            - localhost:9898/readyz
        resources:
          requests:
            cpu: 1m
            memory: 16Mi
---
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: test
  labels:
    app: podinfo
spec:
  type: ClusterIP
  selector:
    app: podinfo
  ports:
  - name: http
    port: 9898
    protocol: TCP
    targetPort: http
```

Create the canary deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo-canary
  namespace: test
  labels:
    app: podinfo-canary
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxUnavailable: 0
    type: RollingUpdate
  selector:
    matchLabels:
      app: podinfo-canary
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
      labels:
        app: podinfo-canary
    spec:
      containers:
      - name: podinfod
        image: quay.io/stefanprodan/podinfo:1.2.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9898
          name: http
          protocol: TCP
        command:
        - ./podinfo
        - --port=9898
        - --level=info
        env:
        - name: PODINFO_UI_COLOR
          value: green
        livenessProbe:
          exec:
            command:
            - podcli
            - check
            - http
            - localhost:9898/healthz
        readinessProbe:
          exec:
            command:
            - podcli
            - check
            - http
            - localhost:9898/readyz
        resources:
          limits:
            cpu: 1000m
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 16Mi
---
apiVersion: v1
kind: Service
metadata:
  name: podinfo-canary
  namespace: test
  labels:
    app: podinfo-canary
spec:
  type: ClusterIP
  selector:
    app: podinfo-canary
  ports:
  - name: http
    port: 9898
    protocol: TCP
    targetPort: http
```

Create the canary horizontal pod auto-scalar:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo-canary
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo-canary
  minReplicas: 2
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 99
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi
```

Create a virtual service (replace `example.com` with your own domain):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  labels:
    app: podinfo
  name: podinfo
  namespace: test
spec:
  gateways:
  - public-gateway.istio-system.svc.cluster.local
  hosts:
  - app.example.com
  - podinfo
  http:
  - route:
    - destination:
        host: podinfo
        port:
          number: 9898
      weight: 100
    - destination:
        host: podinfo-canary
        port:
          number: 9898
      weight: 0
    timeout: 30s
```

Create a rollout custom resource:

```yaml
apiVersion: apps.weave.works/v1beta1
kind: Rollout
metadata:
  name: podinfo
  namespace: test
spec:
  targetKind: Deployment
  primary:
    # deployment name
    name: podinfo
    # clusterIP service name
    host: podinfo
  canary:
    name: podinfo-canary
    host: podinfo-canary
  virtualService:
    name: podinfo
    # used to increment the canary weight
    weight: 10
  metric:
    type: counter
    name: istio_requests_total
    interval: 1m
    # success rate percentage used in canary analysis
    threshold: 99
```

Rollout output:

```
kubectl -n test describe rollout/podinfo

Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    steerer  Starting rollout for podinfo.test
  Normal   Synced  3m    steerer  Advance rollout podinfo.test weight 10
  Normal   Synced  3m    steerer  Advance rollout podinfo.test weight 20
  Normal   Synced  2m    steerer  Advance rollout podinfo.test weight 30
  Normal   Synced  2m    steerer  Advance rollout podinfo.test weight 40
  Normal   Synced  2m    steerer  Advance rollout podinfo.test weight 50
  Normal   Synced  2m    steerer  Advance rollout podinfo.test weight 60
  Normal   Synced  2m    steerer  Advance rollout podinfo.test weight 60
  Warning  Synced  2m    steerer  Halt rollout podinfo.test success rate 88.89% < 99%
  Warning  Synced  2m    steerer  Halt rollout podinfo.test success rate 82.86% < 99%
  Warning  Synced  1m    steerer  Halt rollout podinfo.test success rate 80.49% < 99%
  Warning  Synced  1m    steerer  Halt rollout podinfo.test success rate 82.98% < 99%
  Warning  Synced  1m    steerer  Halt rollout podinfo.test success rate 83.33% < 99%
  Warning  Synced  1m    steerer  Halt rollout podinfo.test success rate 82.22% < 99%
  Warning  Synced  1m    steerer  Halt rollout podinfo.test success rate 94.74% < 99%
  Normal   Synced  1m    steerer  Advance rollout podinfo.test weight 70
  Normal   Synced  55s   steerer  Advance rollout podinfo.test weight 80
  Normal   Synced  45s   steerer  Advance rollout podinfo.test weight 90
  Normal   Synced  35s   steerer  Advance rollout podinfo.test weight 100
  Normal   Synced  25s   steerer  Copying podinfo-canary.test template spec to podinfo.test
  Warning  Synced  15s   steerer  Waiting for podinfo.test rollout to finish: 1 of 2 updated replicas are available
  Normal   Synced  5s    steerer  Promotion complete! Scaling down podinfo-canary.test
```

During the rollout you can generate HTTP 500 errors to test if Steerer pauses the rollout:

```bash
watch -n 1 curl https://app.example.com/status/500
```
