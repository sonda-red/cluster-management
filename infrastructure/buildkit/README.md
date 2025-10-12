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

## Alternative: Using Docker Buildx Kubernetes Driver

If you prefer to use the official Docker Buildx Kubernetes driver instead of the remote endpoint approach, you can create a builder that directly manages BuildKit pods:

```bash
# Create a builder using the Kubernetes driver
docker buildx create \
  --bootstrap \
  --name=sonda-buildkit \
  --driver=kubernetes \
  --driver-opt=namespace=buildkit,rootless=true,replicas=2 \
  --driver-opt=requests.cpu=500m,requests.memory=1Gi \
  --driver-opt=limits.cpu=4000m,limits.memory=8Gi

# Use the builder
docker buildx build \
  --builder=sonda-buildkit \
  -t harbor.sonda.red.intra/my-project/app:latest \
  --push .
```

This approach has the advantage of automatic scaling and load balancing managed by Docker Buildx.

## Troubleshooting

### Common Issues

1. **BuildKit daemon exits with SIGTERM**: This usually indicates the pod is being terminated due to resource constraints or configuration issues. Check the pod logs and ensure proper resource limits are set.

2. **Containerd socket warnings**: The warning `"skipping containerd worker, as \"/run/containerd/containerd.sock\" does not exist"` is expected in rootless mode and can be ignored.

3. **Connection refused errors**: Ensure the BuildKit service is running and the gRPC port (1234) is accessible within the cluster.

### Configuration

The BuildKit daemon is configured via a TOML configuration file mounted from a ConfigMap. Key configurations include:

- **Root directory**: `/home/user/.local/share/buildkit` for buildkit state
- **gRPC address**: `tcp://0.0.0.0:1234` for remote access
- **Worker mode**: OCI worker with overlayfs snapshotter
- **Registry configuration**: Trusted registries including Harbor

### Logs Analysis

If you see logs like:
```
buildkitd: got 1 SIGTERM/SIGINTs, forcing shutdown
error: command [buildkitd] exited: exit status 1
```

This indicates the daemon received a termination signal. Common causes:
- Pod resource limits exceeded
- Kubernetes node issues
- Configuration problems
- Network connectivity issues

Check pod status and logs:
```bash
kubectl get pods -n buildkit
kubectl logs -n buildkit deployment/buildkit
```

## Scaling and Performance

### Multi-replica Setup

For high-load scenarios, you can scale the BuildKit deployment:

```bash
kubectl scale deployment/buildkit -n buildkit --replicas=3
```

The deployment includes:
- **Memory-backed storage**: BuildKit state is stored in memory for faster I/O
- **Resource limits**: Configured for optimal performance with 8GB memory and 4 CPU cores
- **Garbage collection**: Automatic cleanup of build cache to maintain performance
- **Pod disruption budget**: Ensures availability during cluster maintenance

### Monitoring

The BuildKit service exposes a debug endpoint on port 6060 for monitoring:
- Metrics: `http://buildkit.buildkit.svc.cluster.local:6060/debug/metrics`
- Health: `http://buildkit.buildkit.svc.cluster.local:6060/debug/health`

### Performance Optimizations

The configuration includes several performance optimizations:
- **Rootless mode**: Better security without compromising performance
- **Overlayfs snapshotter**: Efficient layer storage
- **Multi-platform support**: Ready for amd64 variants
- **Registry mirrors**: Faster image pulls using mirror.gcr.io for Docker Hub
- **Cache policy**: Intelligent cache management to balance performance and storage

The remote BuildKit service is configured with the same trust bundle that is available to the
runners, so registries backed by the `sonda-red` certificate authority are trusted without any
additional configuration.
