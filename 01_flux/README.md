# Bootstrap Flux

To start our GitOps journey with [Flux](https://fluxcd.io/), we need to bootstrap it first:

```bash
flux check --pre
► checking prerequisites
✔ Kubernetes 1.25.3 >=1.20.6-0
✔ prerequisites checks passed
```

If the checks are successful, you can install Flux on the cluster. First export GitHub credentials:

```bash
export GITHUB_USER=koksay
export GITHUB_TOKEN=<GITHUB_TOKEN>
```

Let’s install Flux on it - if you need to use other options, check out the installation page.

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fast-and-secure \
  --branch=main \
  --path=./gitops/clusters/my-cluster \
  --personal
```

## Install necessary applications

### cert-manager

Create a HelmRepository

```bash
flux create source helm cert-manager \
  --url https://charts.jetstack.io \
  --interval=5m \
  --export > ./gitops/clusters/my-cluster/flux-source-helm-cert-manager-chart.yaml
```

Prepare `values.yaml` for the helm chart:

```bash
cat <<EOF > /tmp/cm-values.yaml
installCRDs: true
EOF
```

Create a HelmRelease

```bash
flux create helmrelease cert-manager \
  --chart cert-manager \
  --source HelmRepository/cert-manager.flux-system \
  --release-name cert-manager \
  --target-namespace cert-manager \
  --create-target-namespace \
  --values /tmp/cm-values.yaml \
  --chart-version 1.13.3 \
  --export > ./gitops/clusters/my-cluster/flux-hr-cert-manager.yaml
```

### ingress-nginx

Create a HelmRepository

```bash
flux create source helm ingress-nginx \
  --url https://kubernetes.github.io/ingress-nginx \
  --interval=5m \
  --export > ./gitops/clusters/my-cluster/flux-source-helm-ingress-nginx-chart.yaml
```

Prepare `values.yaml` for the helm chart:

```bash
# Get the Ingress IP address
export INGRESS_IP=$(gcloud compute addresses list --filter="region:europe-west1" --filter="name=fast-and-secure-addr" --format="get(address)")

cat <<EOF > /tmp/ingress-values.yaml
controller:
  service:
    loadBalancerIP: ${INGRESS_IP}
EOF
```

Create a HelmRelease

```bash
flux create helmrelease ingress-nginx \
  --chart ingress-nginx \
  --source HelmRepository/ingress-nginx.flux-system \
  --release-name ingress-nginx \
  --target-namespace ingress-nginx \
  --create-target-namespace \
  --values /tmp/ingress-values.yaml \
  --chart-version 4.8.3 \
  --export > ./gitops/clusters/my-cluster/flux-hr-ingress-nginx.yaml
```

### Deploy

Commit changes and see the GitOps magic:

```bash
## first pull the changes by flux bootstrapping
git pull

git add .
git commit -am "Deploy nginx and cert-manager"
git push

## wait and check:
kubectl get helmrepo,hr -n flux-system

kubectl wait hr cert-manager -n flux-system --for=condition=ready --timeout=600s
kubectl wait hr ingress-nginx -n flux-system --for=condition=ready --timeout=600s
```
