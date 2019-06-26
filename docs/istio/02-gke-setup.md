# GKE cluster setup

Create a cluster with three nodes using the latest Kubernetes version:

```bash
k8s_version=$(gcloud container get-server-config --format=json \
| jq -r '.validNodeVersions[0]')

gcloud container clusters create istio \
--cluster-version=${k8s_version} \
--zone=europe-west3-a \
--num-nodes=3 \
--machine-type=n1-highcpu-4 \
--preemptible \
--no-enable-cloud-logging \
--disk-size=50 \
--enable-autorepair \
--scopes=gke-default
```

The above command will create a default node pool consisting of `n1-highcpu-4` (vCPU: 4, RAM 3.60GB, DISK: 30GB) preemptible VMs.
Preemptible VMs are up to 80% cheaper than regular instances and are terminated and replaced after a maximum of 24 hours.

Set up credentials for `kubectl`:

```bash
gcloud container clusters get-credentials istio -z=europe-west3-a
```

Create a cluster admin role binding:

```bash
kubectl create clusterrolebinding "cluster-admin-$(whoami)" \
--clusterrole=cluster-admin \
--user="$(gcloud config get-value core/account)"
```

Validate your setup with:

```bash
kubectl get nodes -o wide
```

Next: [Cloud DNS setup](03-clouddns-setup.md)
