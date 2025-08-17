# sonda-red/cluster-management

This repository contains the complete GitOps-driven management for the sonda.red homelab. All infrastructure, platform, and application deployments are managed as code using Flux, Kustomize, and Helm.

## Features

- **GitOps with Flux:** Automated reconciliation of cluster state from Git, including SOPS-encrypted secrets.
- **Secure Secrets:** SOPS is used for secret encryption, with decryption handled by Flux in-cluster.

## Repository Structure

```
├── bootstrap/           # Initial setup scripts and Helmfile definitions
├── clusters/            # Cluster-specific overlays and Flux Kustomizations
├── infrastructure/      # Core and platform components (networking, storage, monitoring, etc.)
├── reserve/             # Reserved resources for future expansion
├── templates/           # Reusable manifest and HelmRelease templates
├── renovate.json        # Renovate configuration for automated dependency updates
└── ...
```

## Key Components

- **MinIO**: S3-compatible object storage for backups and as a storage backend for other services.
- **Harbor**: Private container registry for storing and managing container images.
- **PostgreSQL**: Managed database service for stateful workloads.
- **Redis**: In-memory data store, often used for caching or as a message broker.
- **kube-prometheus-stack**: Prometheus for metrics, Alertmanager for notifications, and Grafana for dashboards.
- **VictoriaLogs**: Log aggregation and storage, integrated with Grafana for log exploration.
- **Node Feature Discovery (NFD)**: Detects hardware features and labels nodes for scheduling.
- **MetalLB**: Load balancer for bare-metal Kubernetes clusters.
- **cert-manager**: Automated management and issuance of TLS certificates.
- **Ingress-NGINX**: Ingress controller for HTTP routing.
- **System Upgrade Controller**: Automates k3s upgrades and node maintenance.
- **Intel Device Operator**: Manages Intel hardware plugins for Kubernetes.
- **Actions Runner Controller**: Manages self-hosted GitHub Actions runners in Kubernetes.

## Getting Started

1. **Clone the repository:**
   ```sh
   git clone https://github.com/sonda-red/cluster-management.git
   ```
2. **Review the `clusters/` directory** for your cluster's entrypoint and overlays.
3. **Bootstrap Flux** using the manifests in `bootstrap/` or your preferred method.
4. **Configure secrets:** Encrypt your secrets with SOPS and commit them to the repo.
5. **Monitor Flux:** Flux will reconcile the cluster state automatically.

## Security
- All sensitive data is encrypted with SOPS.
- Only Flux in the cluster can decrypt and apply secrets.

## Automated Updates
- [Renovate](https://github.com/renovatebot/renovate) is configured to keep Helm releases and dependencies up to date.
