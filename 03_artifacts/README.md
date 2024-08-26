# Use OCI Registry for Airgapped Deployments

## Create a Local Minikube cluster and Deploy zot

```bash
minikube start --cpus=2 --memory=4096

helm upgrade --install zot zot \
  --repo http://zotregistry.dev/helm-charts \
  --version 0.1.60
```

Port-forward to Zot and check nothing exists on [the UI](http://localhost:5000)

```bash
kubectl port-forward svc/zot 5000
```

## Artifact Mamagement

Get the `webserver` artifact from our installation on the cloud via `oras`:

```bash
oras copy -r \
  --from-distribution-spec v1.1-referrers-api \
  --to-distribution-spec v1.1-referrers-tag \
  fast-and-secure.k8c.io/nginx:0.1.0 \
  --to-oci-layout webserver:0.1.0
```

This will create a directory called webserver, check it out:

```bash
tree webserver
```

Copy those files to the Zot registry running on the Minikube locally:

```bash
oras copy -r \
  --from-oci-layout \
  ./webserver:0.1.0  \
  localhost:5000/webserver:0.1.0
```

Check [the UI](http://localhost:5000) again and see the signed `webserver` package.

Verfy the package again:

```bash
cosign verify \
  --certificate-identity koray.oksay@gmail.com \
  --certificate-oidc-issuer https://github.com/login/oauth \
  --experimental-oci11=true \
  localhost:5000/webserver:0.1.0
```

Install it:

```bash
helm upgrade --install \
  webserver oci://localhost:5000/webserver \
  --version 0.1.0
```
