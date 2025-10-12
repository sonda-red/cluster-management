# BuildKit Service

This directory deploys a shared [BuildKit](https://github.com/moby/buildkit) instance that can be
used by the self-hosted GitHub runners. The service runs the rootless BuildKit image, exposes the
gRPC endpoint on port `1234`, and installs the `sonda-red` trust bundle so that image pushes to
internal registries such as Harbor continue to work.

## Connecting from GitHub Actions

The BuildKit deployment is exposed inside the cluster at
`buildkit.buildkit.svc.cluster.local:1234`. Runners can connect to it by configuring buildx to use a
remote endpoint. The example workflow below shows how to authenticate to Harbor, create a remote
builder that targets the shared BuildKit instance, and build an image:

```yaml
name: Build container with remote BuildKit

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Harbor
        uses: docker/login-action@v3
        with:
          registry: harbor.sonda.red.intra
          username: ${{ secrets.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      - name: Set up buildx for the shared BuildKit service
        id: buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver: remote
          endpoint: tcp://buildkit.buildkit.svc.cluster.local:1234

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          builder: ${{ steps.buildx.outputs.name }}
          context: .
          push: true
          tags: harbor.sonda.red.intra/my-project/app:latest
```

The remote BuildKit service is configured with the same trust bundle that is available to the
runners, so registries backed by the `sonda-red` certificate authority are trusted without any
additional configuration.
