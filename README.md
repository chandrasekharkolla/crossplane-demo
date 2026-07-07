# EKS with Crossplane — Local Platform Engineering Demo

A complete local platform engineering stack. One command spins up a kind cluster with ArgoCD managing everything via GitOps: Crossplane, Floci (local AWS emulator), Crossview, and Backstage. No AWS account, no credentials, no cloud costs.

## What's Inside

- **ArgoCD** — GitOps controller managing all components via app-of-apps
- **Crossplane** — infrastructure control plane with AWS providers
- **Floci** — local AWS emulator running as a pod
- **floci-dash** — AWS Console-style dashboard to visualize AWS resources
- **Crossview** — Crossplane dashboard for XRDs, Compositions, and Claims
- **Backstage** — developer portal with Kubernetes plugin
- **EKS Composition** — platform API that creates EKS clusters from a simple Claim

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kind](https://kind.sigs.k8s.io/) — `brew install kind`
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — `brew install kubectl`
- [Task](https://taskfile.dev/) — `brew install go-task`

## Quick Start

1. Clone the repo:

```
git clone <repo-url>
cd crossplane-demo
```

2. Run setup:

```
task setup
```

This creates a kind cluster, installs ArgoCD, and deploys the app-of-apps. ArgoCD then syncs everything else automatically. The repo URL is auto-detected from your git remote.

Already have a cluster? Just point your kubeconfig at it and run:

```
task install
```

This skips kind creation and installs directly on whatever cluster you're connected to (EKS, k3s, Docker Desktop, etc.).

## Usage

```
task setup             # New kind cluster + ArgoCD + app-of-apps
task install           # Install on an existing cluster (skip kind)
task demo              # Create a sample EKS cluster from a Claim
task status            # Show Claim status and resource tree
task check             # Verify all components are healthy
task uninstall         # Remove platform from cluster (keeps cluster)
task teardown          # Delete the kind cluster (removes everything)
```

### Dashboards

```
task open:argocd        # ArgoCD UI        — https://localhost:8443
task argocd:password    # Get admin password
task open:dashboard     # floci-dash       — http://localhost:9877
task open:crossview     # Crossview        — http://localhost:8080
task open:backstage     # Backstage        — http://localhost:7007
```

## The Claim

What a developer writes to get a fully configured EKS cluster:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: EKSCluster
metadata:
  name: my-cluster
spec:
  version: "1.29"
  nodeSize: medium    # small | medium | large
  nodeCount: 3        # 1-10
```

This creates 5 AWS resources: EKS Cluster, Managed Node Group, IAM Cluster Role, IAM Node Role, and Security Group.

## Architecture

```
Taskfile bootstraps          ArgoCD manages (GitOps)
─────────────────           ──────────────────────────
kind cluster         ──►    Crossplane          (Helm, wave 0)
ArgoCD               ──►    AWS Providers       (manifests, wave 1)
app-of-apps          ──►    Floci + floci-dash  (manifests, wave 2)
                            Crossview           (Helm, wave 2)
                            Backstage           (manifests, wave 2)
                            Platform XRD/Comp   (manifests, wave 3)
```

SyncWaves ensure correct ordering: Crossplane CRDs exist before providers install, providers are healthy before platform XRDs apply.

```
kind cluster
├── argocd/              ArgoCD + app-of-apps
├── crossplane-system/   Crossplane + AWS providers
├── crossview/           Crossplane dashboard
├── floci/               Floci (AWS emulator) + floci-dash
├── backstage/           Developer portal
└── default/             EKSCluster Claims
```

Four dashboards, four perspectives:
- ArgoCD (localhost:8443) — GitOps sync status for all components
- floci-dash (localhost:9877) — AWS resources Crossplane created
- Crossview (localhost:8080) — XRDs, Compositions, Claims, managed resources
- Backstage (localhost:7007) — developer portal with service catalog

## Repo Structure

```
├── Taskfile.yml              # Bootstrap: kind + ArgoCD + app-of-apps
├── argocd/
│   ├── app-of-apps.yaml      # Root ArgoCD Application
│   └── crossplane-compat.yaml
├── apps/                      # ArgoCD Application definitions
│   ├── crossplane.yaml        # Wave 0 — Helm chart
│   ├── providers.yaml         # Wave 1 — providers + functions + config
│   ├── floci.yaml             # Wave 2 — Floci + floci-dash
│   ├── crossview.yaml         # Wave 2 — Helm chart
│   ├── backstage.yaml         # Wave 2 — developer portal
│   └── platform.yaml          # Wave 3 — XRD + Composition
├── crossplane/                # Synced by ArgoCD
│   ├── providers.yaml
│   ├── functions.yaml
│   ├── credentials.yaml
│   └── providerconfig.yaml
├── floci/                     # Synced by ArgoCD
│   ├── deployment.yaml
│   └── service.yaml
├── backstage/                 # Synced by ArgoCD
│   ├── deployment.yaml
│   └── service.yaml
├── platform/                  # Synced by ArgoCD
│   ├── xrd.yaml
│   └── composition.yaml
├── kind/
│   └── cluster.yaml
└── examples/
    └── claim.yaml
```

## How It Works

1. `task setup` creates a kind cluster and installs ArgoCD
2. The app-of-apps Application tells ArgoCD to sync everything in `apps/`
3. Each Application in `apps/` points at either a Helm chart or a directory in this repo
4. SyncWaves ensure correct ordering (Crossplane before providers before platform)
5. ArgoCD continuously reconciles — change a file in Git, ArgoCD syncs it
6. `task demo` applies a Claim that Crossplane processes against the local Floci emulator

## Limitations

- Floci emulates AWS APIs, not full service behavior — EKS won't schedule pods
- Backstage uses the community image with default config — customize for your org
- Some Crossplane status fields may return placeholder values from Floci
- AWS credentials in `crossplane/credentials.yaml` are dummy values for Floci (not real keys)
