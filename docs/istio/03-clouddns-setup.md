# Cloud DNS setup

You will need an internet domain and access to the registrar to change the name servers to Google Cloud DNS.

Create a managed zone named `istio` in Cloud DNS (replace `example.com` with your domain):

```bash
gcloud dns managed-zones create \
--dns-name="example.com." \
--description="Istio zone" "istio"
```

Look up your zone's name servers:

```bash
gcloud dns managed-zones describe istio
```

Update your registrar's name server records with the records returned by the above command. 

Wait for the name servers to change (replace `example.com` with your domain):

```bash
wait dig +short NS example.com
```

Create a static IP address named `istio-gateway-ip` in the same region as your GKE cluster:

```bash
gcloud compute addresses create istio-gateway-ip --region europe-west3-a
```

Find the static IP address:

```bash
gcloud compute addresses describe istio-gateway-ip --region europe-west3-a
```

Create the following DNS records (replace `example.com` with your domain and set your Istio Gateway IP):

```bash
DOMAIN="example.com"
GATEWAYIP="35.198.98.90"

gcloud dns record-sets transaction start --zone=istio

gcloud dns record-sets transaction add --zone=istio \
--name="${DOMAIN}" --ttl=300 --type=A ${GATEWAYIP}

gcloud dns record-sets transaction add --zone=istio \
--name="www.${DOMAIN}" --ttl=300 --type=CNAME ${DOMAIN}

gcloud dns record-sets transaction add --zone=istio \
--name="*.${DOMAIN}" --ttl=300 --type=A ${GATEWAYIP}

gcloud dns record-sets transaction execute --zone istio
```

Verify that the wildcard DNS is working (replace `example.com` with your domain):

```bash
watch host test.example.com
```
