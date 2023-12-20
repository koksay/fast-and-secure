# Deploy the Registry and Start Using it

## ClusterIssuer

Create a `ClusterIssuer` for the Ingress:

```bash
cat <<EOF > ./gitops/clusters/my-cluster/cert-manager-cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

## Deploy zot

Create a HelmRepository

```bash
flux create source helm zot \
  --url http://zotregistry.io/helm-charts \
  --interval=5m \
  --export > ./gitops/clusters/my-cluster/flux-source-helm-zot-chart.yaml
```

Prepare `values.yaml` for the helm release:

```bash
cat <<EOF > /tmp/zot-values.yaml
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
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
```

Create a HelmRelease

```bash
flux create helmrelease zot \
  --chart zot \
  --source HelmRepository/zot.flux-system \
  --release-name zot \
  --target-namespace zot \
  --create-target-namespace \
  --values /tmp/zot-values.yaml \
  --chart-version 0.1.42 \
  --export > ./gitops/clusters/my-cluster/flux-hr-zot.yaml
```

Push to Git:

```bash
git add .
git commit -am "Deploy zot registry"
git push
```

## Create, sign, push, and use a Helm Chart

Create a directory for the chart

```bash
mkdir -p /tmp/helm-oci-demo
cd /tmp/helm-oci-demo
```

Create an example chart

```bash
helm create webserver
helm package webserver
```

Push the helm chart to the OCI registry

```bash
helm push webserver-0.1.0.tgz oci://fast-and-secure.k8c.io
# output:
# Pushed: fast-and-secure.k8c.io/webserver:0.1.0
# Digest: sha256:557c75156abce75d720f5c1f8d90dec1e1cc9c665c17865373374ab4794186a0
```

Check the chart on the [Zot UI](https://fast-and-secure.k8c.io/)

Sign with cosign.

> Select GitHub on the opening browser window to login to sigstore.

```bash
cosign sign fast-and-secure.k8c.io/webserver:0.1.0

# even better, sigh with SHA256:
cosign sign fast-and-secure.k8c.io/webserver@sha256:557c75156abce75d720f5c1f8d90dec1e1cc9c665c17865373374ab4794186a0

# output:
# Generating ephemeral keys...
# Retrieving signed certificate...
#
#        The sigstore service, hosted by sigstore a Series of LF Projects, LLC, is provided pursuant to the Hosted Project Tools Terms of Use, available at https://lfprojects.org/policies/hosted-project-tools-terms-of-use/.
#        Note that if your submission includes personal data associated with this signed artifact, it will be part of an immutable record.
#        This may include the email address associated with the account with which you authenticate your contractual Agreement.
#        This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later, and is subject to the Immutable Record notice at https://lfprojects.org/policies/hosted-project-tools-immutable-records/.

# By typing 'y', you attest that (1) you are not submitting the personal data of any other person; and (2) you understand and agree to the statement and the Agreement terms at the URLs listed above.
# Are you sure you would like to continue? [y/N] y
# Your browser will now be opened to:
# https://oauth2.sigstore.dev/auth/auth?access_type=online&client_id=sigstore&code_challenge=kGaDDKFtPBswqbt2XLP0vG8eatOcX-h2qIJywmWengU&code_challenge_method=S256&nonce=2ZnT5VupQNksB4Tu6gfMiiPKWgp&redirect_uri=http%3A%2F%2Flocalhost%3A42617%2Fauth%2Fcallback&response_type=code&scope=openid+email&state=2ZnT5UuIYnWtsTwv7fhVqZZyyuY
# Successfully verified SCT...
# WARNING: Image reference fast-and-secure.k8c.io/webserver:0.1.0 uses a tag, not a digest, to identify the image to sign.
#    This can lead you to sign a different image than the intended one. Please use a
#    digest (example.com/ubuntu@sha256:abc123...) rather than tag
#    (example.com/ubuntu:latest) for the input to cosign. The ability to refer to
#    images by tag will be removed in a future release.

# tlog entry created with index: 58132223
# Pushing signature to: fast-and-secure.k8c.io/webserver
```

Verify with cosign:

```bash
cosign verify \
  --certificate-identity koray.oksay@gmail.com \
  --certificate-oidc-issuer https://github.com/login/oauth \
  fast-and-secure.k8c.io/webserver:0.1.0
```

Check again the chart on the [Zot UI](https://fast-and-secure.k8c.io/)

### Deploy with Flux

Now install Webserver from our registry

```bash
flux create source helm demo-charts\
    --url=oci://fast-and-secure.k8c.io \
    --interval=5m \
    --export> ./gitops/clusters/my-cluster/flux-source-helm-webserver.yaml

flux create helmrelease webserver \
    --source=HelmRepository/demo-charts \
    --chart=webserver \
    --interval=5m \
    --release-name webserver \
    --target-namespace=default \
    --export | yq '.spec.chart.spec|=({"verify": { "provider": "cosign" } } +.)' > ./gitops/clusters/my-cluster/flux-hr-webserver.yaml
```

Update git repo and watch flux magic

```bash
git add .
git commit -am "Add Webserver deployment"
git push

flux get helmreleases -A
```

When the chart is deployed, make sure it's verified:

```bash
kubectl get helmchart flux-system-webserver -n flux-system \
  -ojsonpath='{.status.conditions[?(@.type=="SourceVerified")]}' | jq
```
