# ArgoCD GitOps Setup

Complete ArgoCD deployment for managing your Kubernetes applications via GitOps.

## Features

- **ArgoCD**: Declarative GitOps continuous delivery
- **App of Apps Pattern**: Manage multiple applications from one place
- **SOPS Integration**: Encrypted secrets in Git
- **Project Management**: Organize applications by project
- **RBAC**: Role-based access control
- **SSO Ready**: OIDC/LDAP integration examples
- **Notifications**: Slack/Discord alerts for sync events

## Quick Start

### Prerequisites

- Kubernetes cluster (1.24+)
- kubectl configured
- Git repository for storing manifests

### Install ArgoCD

```bash
# Method 1: Quick install (for testing)
kubectl apply -k base

# Method 2: With custom configuration
kubectl apply -k overlays/local-path

# Wait for ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### Access ArgoCD

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access: https://localhost:8080
# Username: admin
# Password: (from above)
```

### Deploy Your First Application

```bash
# Deploy monitoring stack
kubectl apply -f applications/monitoring.yaml

# Deploy GitLab
kubectl apply -f applications/gitlab.yaml

# Or use CLI
argocd app create monitoring \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path "Prometheus & Grafana Setup/overlays/local-path" \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --sync-policy automated
```

## What Gets Deployed

### Core Components

- **argocd-server**: Web UI and API server
- **argocd-repo-server**: Repository service
- **argocd-application-controller**: Application controller
- **argocd-dex-server**: SSO/OAuth (optional)
- **argocd-redis**: Cache and session storage

### Storage

- **Redis**: emptyDir (ephemeral)
- **Repo Server**: emptyDir (ephemeral, repos cloned on demand)

### Resource Allocation

- **argocd-server**: 100m CPU, 128Mi memory
- **argocd-repo-server**: 100m CPU, 256Mi memory
- **argocd-application-controller**: 250m CPU, 512Mi memory
- **argocd-redis**: 50m CPU, 64Mi memory

**Total**: ~500m CPU, ~1Gi memory

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                          Git Repository                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Monitoring  │  │   GitLab    │  │    Apps     │        │
│  │   Stack     │  │             │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ Git Pull
                           ▼
              ┌────────────────────────┐
              │     ArgoCD Server      │
              │  ┌──────────────────┐  │
              │  │  Application     │  │
              │  │  Controller      │  │
              │  └──────────────────┘  │
              └────────────┬───────────┘
                           │
                           │ Kubectl Apply
                           ▼
              ┌────────────────────────┐
              │   Kubernetes Cluster   │
              │  ┌──────────────────┐  │
              │  │   Applications   │  │
              │  │                  │  │
              │  │ • Monitoring     │  │
              │  │ • GitLab         │  │
              │  │ • Apps           │  │
              │  └──────────────────┘  │
              └────────────────────────┘
```

## Application Structure

### App of Apps Pattern

```
applications/
├── root.yaml              # Root app that manages all apps
├── monitoring.yaml        # Monitoring stack
├── gitlab.yaml           # GitLab
├── infrastructure/       # Core infrastructure
│   ├── ingress.yaml
│   ├── cert-manager.yaml
│   └── longhorn.yaml
└── apps/                 # Your applications
    ├── linkding.yaml
    └── homepage.yaml
```

### Deploy All Apps

```bash
# Create root app (app of apps)
kubectl apply -f applications/root.yaml

# ArgoCD will automatically create and sync all child apps
```

## SOPS Integration

### Setup SOPS with ArgoCD

```bash
# 1. Create age key for SOPS
age-keygen -o ~/.config/sops/age/keys.txt

# 2. Create secret in ArgoCD namespace
kubectl create secret generic sops-age-key \
  -n argocd \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt

# 3. Install SOPS plugin
kubectl apply -f examples/sops-plugin.yaml

# 4. Restart ArgoCD repo server
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

### Use SOPS-Encrypted Secrets

```bash
# Encrypt secret
sops -e -i ../monitoring/base/grafana/secret.yaml

# Commit to Git
git add ../monitoring/base/grafana/secret.yaml
git commit -m "Add encrypted secrets"

# ArgoCD will automatically decrypt during sync
```

## Projects & RBAC

### Create Project

```bash
# Apply project definition
kubectl apply -f examples/project-homelab.yaml

# Projects organize and restrict app deployment
```

### Configure RBAC

See `examples/rbac-policy.yaml` for role configuration.

## Notifications

### Discord Notifications

```bash
# Configure Discord webhook
kubectl apply -f examples/notifications-discord.yaml

# Notifications will be sent on:
# - Sync started/completed/failed
# - App health degraded
# - App out of sync
```

### Slack Notifications

See `examples/notifications-slack.yaml`

## Best Practices

### 1. Use App of Apps Pattern

Organize apps hierarchically for easier management.

### 2. Enable Auto-Sync

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 3. Use Sync Waves

Control deployment order:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy first
```

### 4. Health Checks

Define custom health checks for your CRDs:

```yaml
resource.customizations: |
  argoproj.io/Application:
    health.lua: |
      # Custom health check logic
```

### 5. Ignore Differences

Ignore auto-generated fields:

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas  # Ignore HPA changes
```

## CLI Usage

### Install ArgoCD CLI

```bash
# Arch Linux
sudo pacman -S argocd

# Or download binary
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

### Common Commands

```bash
# Login
argocd login localhost:8080

# List apps
argocd app list

# Get app details
argocd app get monitoring

# Sync app
argocd app sync monitoring

# View sync status
argocd app wait monitoring --health

# View logs
argocd app logs monitoring

# Delete app
argocd app delete monitoring

# Create app
argocd app create myapp \
  --repo https://github.com/user/repo.git \
  --path overlays/prod \
  --dest-namespace myapp \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated
```

## Troubleshooting

### Application Stuck in Progressing

```bash
# Check app status
argocd app get APP_NAME

# View sync result
kubectl describe application APP_NAME -n argocd

# Check controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Sync Fails with Permission Error

```bash
# Check service account
kubectl get sa -n argocd

# Verify RBAC
kubectl auth can-i create deployments --as=system:serviceaccount:argocd:argocd-application-controller -n TARGET_NAMESPACE
```

### SOPS Decryption Fails

```bash
# Check secret exists
kubectl get secret sops-age-key -n argocd

# Check age key format
kubectl get secret sops-age-key -n argocd -o jsonpath='{.data.keys\.txt}' | base64 -d

# Check repo server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=50
```

## Documentation

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [SOPS Plugin](https://github.com/viaduct-ai/kustomize-sops)
- [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

## Integration with Existing Setup

### Deploy Monitoring Stack

```bash
kubectl apply -f applications/monitoring.yaml
```

### Deploy GitLab

```bash
kubectl apply -f applications/gitlab.yaml
```

### Deploy Everything

```bash
kubectl apply -f applications/root.yaml
```

ArgoCD will manage your entire cluster state via Git!
