# Prerequisites

You will be creating a cluster on Google’s Kubernetes Engine (GKE), 
if you don’t have an account you can sign up [here](https://cloud.google.com/free/) for free credits.

Login into GCP, create a project and enable billing for it. 

Install the [gcloud](https://cloud.google.com/sdk/) command line utility and configure your project with `gcloud init`.

Set the default project (replace `PROJECT_ID` with your own project):

```bash
gcloud config set project PROJECT_ID
```

Set the default compute region and zone:

```bash
gcloud config set compute/region europe-west3
gcloud config set compute/zone europe-west3-a
```

Enable the Kubernetes and Cloud DNS services for your project:

```bash
gcloud services enable container.googleapis.com
gcloud services enable dns.googleapis.com
```

Install the `kubectl` command-line tool:

```bash
gcloud components install kubectl
```

Install the `helm` command-line tool:

```bash
brew install kubernetes-helm
```

Next: [GKE cluster setup](02-gke-setup.md)
