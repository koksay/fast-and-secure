# Deploy the Registry and Start Using it

## Deploy zot

Create a HelmRepository

```bash
flux create source helm zot \
  --url http://zotregistry.io/helm-charts \
  --interval=5m \
  --export > ./gitops/clusters/my-cluster/flux-source-helm-zot-chart.yaml
```

Create a HelmRelease

```bash
cat <<EOF > /tmp/zot-values.yaml
ingress:
  enabled: true
  annotations:
    cert-manager.io/issuer: "letsencrypt"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  className: "nginx"
  pathtype: ImplementationSpecific
  hosts:
    - host: fast-and-secure.k8c.io
      paths:
        - path: /
  tls:
    - secretName: fast-and-secure
      hosts:
        - fast-and-secure.k8c.io
EOF

flux create helmrelease zot \
  --chart zot \
  --source HelmRepository/zot.flux-system \
  --release-name zot \
  --target-namespace zot \
  --create-target-namespace \
  --values /tmp/zot-values.yaml \
  --chart-version 0.1.40 \
  --export > ./gitops/clusters/my-cluster/flux-hr-zot.yaml
```

## Create, sign, push, and use a Helm Chart

Create a directory for the chart

```bash
mkdir -p helm-oci-demo
cd helm-oci-demo
```

Create an example chart

```bash
helm create nginx
helm package nginx
```

Push the helm chart to the OCI registry

```bash
helm push nginx-0.1.0.tgz oci://fast-and-secure.k8c.io
# output:
# Pushed: 178.62.227.10:30725/nginx:0.1.0
# Digest: sha256:557c75156abce75d720f5c1f8d90dec1e1cc9c665c17865373374ab4794186a0
```

Sign and verify with cosign

```bash
$> cosign generate-key-pair
$> cosign sign -key cosign.key \
   fast-and-secure.k8c.io/nginx@sha256:557c75156abce75d720f5c1f8d90dec1e1cc9c665c17865373374ab4794186a0 


## verify
cosign verify --key cosign.key \
   fast-and-secure.k8c.io/nginx@sha256:557c75156abce75d720f5c1f8d90dec1e1cc9c665c17865373374ab4794186a0

kubectl -n flux-system create secret generic cosign-pub \
  --from-file=cosign.pub=cosign.pub
```

Now install NGINX from our registry

```bash
flux create source helm demo-charts\
    --url=oci://fast-and-secure.k8c.io \
    --interval=5m \
    --export> ./clusters/my-cluster/flux-source-helm-nginx.yaml

flux create helmrelease nginx \
    --source=HelmRepository/demo-charts \
    --chart=nginx \
    --interval=5m \
    --release-name nginx \
    --target-namespace=default \
    --export | yq '.spec.chart.spec|=({"verify": { "provider": "cosign", "secretRef": { "name": "cosign-pub" } } } +.)' > ./clusters/my-cluster/flux-hr-nginx.yaml
```

Update git repo and watch flux magic

```bash
git add .
git commit -am "Add NGINX deployment"
git push

flux get helmreleases -A
```
