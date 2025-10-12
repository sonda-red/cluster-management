# BuildKit service

This directory deploys a singleton [BuildKit](https://github.com/moby/buildkit)
daemon that can be used by the GitHub ARC runners to perform container builds
without relying on DinD sidecars.

## Architecture

* Runs in the `buildkit` namespace with a dedicated service account.
* Exposes the BuildKit TCP endpoint on port `1234` via the `buildkit`
  `ClusterIP` service (`tcp://buildkit.buildkit.svc.cluster.local:1234`).
* Mounts the `sonda-red-intra-root-ca` trust bundle that is distributed by the
  cert-manager trust-manager so the daemon trusts internal registries such as
  `harbor.sonda.red.intra`.

## Example GitHub workflow

The following workflow demonstrates how to target the remote builder from a job
executing on the Kubernetes ARC runners. The workflow logs in to Harbor using
`docker/login-action`, configures Buildx in remote mode and pushes an image.

```yaml
name: Build with BuildKit

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on:
      - self-hosted
      - arc-set-nuc
    container:
      image: docker:24.0-cli
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Buildx (remote BuildKit)
        uses: docker/setup-buildx-action@v3
        with:
          driver: remote
          endpoint: tcp://buildkit.buildkit.svc.cluster.local:1234

      - name: Authenticate to Harbor
        uses: docker/login-action@v3
        with:
          registry: harbor.sonda.red.intra
          username: ${{ secrets.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: harbor.sonda.red.intra/library/my-app:${{ github.sha }}
```

The ARC runner pods automatically mount the same trust bundle that BuildKit
uses, so registry communication from both the runner and BuildKit respects the
cluster's certificate authority.
