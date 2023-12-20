# Setup

In this task, we will setup the needed components for the demo.

## Create Kubernetes Cluster

```bash
./00_setup/setup_cluster.sh
```

## Store the IP Adress of the Ingress LB in an environment variable

```bash
export INGRESS_IP=$(gcloud compute addresses list --filter="region:europe-west1" \
  --filter="name=fast-and-secure-addr" --format="get(address)")
```

## DNS entry

Make sure that you entered A record for your domain. For example: `fast-and-secure.k8c.io IN A ${INGRESS_IP}`
