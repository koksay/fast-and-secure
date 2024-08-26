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
  fast-and-secure.k8c.io/webserver:0.1.0 \
  --to-oci-layout webserver:0.1.0
```

This will create a directory called webserver, check it out:

```bash
tree webserver

# output:
webserver
├── blobs
│   └── sha256
│       ├── 56c54983b377cdfde0e8442351cd18f543948d40b90ad1b5f2cd2343bfc2e4aa
│       ├── 58902d00c6eb919ecdb4a75dc6abba031b96eb38386ab2991649d97cf7ec15cc
│       ├── 591533f427c3730f1bf39653436b6e3f4b24877ef32ee9bc88a4ec7a4507e513
│       ├── 60e7616d6ee3aea4b0c35ba8ba5a1b51d54f4efda35155aa20f69d634f20a680
│       ├── 8dd947c660f84580e55b2430db37eb4bb7506cc577a491a8d76b39c2867ead9b
│       └── eeff84dc9db53ef12374ff34def8f8e576014a79dba2d4e17c001671d5748a2e
├── index.json
├── ingest
└── oci-layout
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
