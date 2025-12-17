# ArgoCD GitOps Setup

[![Lines of Code](https://tokei.rs/b1/github/USERNAME/REPO-NAME)](https://github.com/USERNAME/REPO-NAME)
[![License](https://img.shields.io/github/license/USERNAME/REPO-NAME)](https://github.com/USERNAME/REPO-NAME/blob/master/LICENSE)
[![GitHub last commit](https://img.shields.io/github/last-commit/aaronwittchen/ArgoCD?logo=github)](https://github.com/aaronwittchen/ArgoCD/commits/master)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/5d4efbb0d525449c810ed5427a6910d5)](https://app.codacy.com/gh/aaronwittchen/ArgoCD/dashboard?utm_source=gh&utm_medium=referral&utm_content=&utm_campaign=Badge_grade)
[![CodeFactor](https://www.codefactor.io/repository/github/aaronwittchen/argocd/badge)](https://www.codefactor.io/repository/github/aaronwittchen/argocd)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=aaronwittchen_ArgoCD&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=aaronwittchen_ArgoCD)
[![Maintainability Rating](https://sonarcloud.io/api/project_badges/measure?project=aaronwittchen_ArgoCD&metric=sqale_rating)](https://sonarcloud.io/summary/new_code?id=aaronwittchen_ArgoCD)
[![Reliability Rating](https://sonarcloud.io/api/project_badges/measure?project=aaronwittchen_ArgoCD&metric=reliability_rating)](https://sonarcloud.io/summary/new_code?id=aaronwittchen_ArgoCD)
[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=aaronwittchen_ArgoCD&metric=security_rating)](https://sonarcloud.io/summary/new_code?id=aaronwittchen_ArgoCD)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=aaronwittchen_ArgoCD&metric=vulnerabilities)](https://sonarcloud.io/summary/new_code?id=aaronwittchen_ArgoCD)

Complete ArgoCD deployment for managing your Kubernetes applications via GitOps.

## Features

- **ArgoCD**: Declarative GitOps continuous delivery
- **App of Apps Pattern**: Manage multiple applications from one place
- **SOPS Integration**: Encrypted secrets in Git
- **Project Management**: Organize applications by project
- **RBAC**: Role-based access control
- **SSO Ready**: OIDC/LDAP integration examples
- **Notifications**: Slack/Discord alerts for sync events

This setup includes SOPS encryption for secrets. You **MUST** follow these security practices:

- Always create the sops-age-key secret manually (see SOPS Integration section)
- Enable TLS for production deployments (cert-manager recommended)
- Review sync policies before enabling automated sync in production

## Quick Start

### Install ArgoCD

```bash
# Step 1: Generate AGE key (if you don't have one)
age-keygen -o ~/.config/sops/age/keys.txt

# Step 2: Create the SOPS secret in ArgoCD namespace
kubectl create namespace argocd
kubectl create secret generic sops-age-key \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt \
  -n argocd

# Step 3: Apply ArgoCD (choose your storage overlay)
# For local-path provisioner:
kubectl apply -k overlays/local-path

# For Longhorn:
kubectl apply -k overlays/longhorn

# Step 4: Wait for ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### Access ArgoCD

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward UI (server runs in insecure/HTTP mode)
kubectl port-forward svc/argocd-server -n argocd 8080:80

# Access: http://localhost:8080
# Or via Gateway: http://argo.k8s.home
# Username: admin
# Password: (from above)
```

### Deploy Your First Application

```bash
# Deploy monitoring stack
kubectl apply -f applications/monitoring.yaml

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

- **argocd-server**:
- **argocd-repo-server**:
- **argocd-application-controller**:
- **argocd-redis**:

**Total**:

### Deploy All Apps

```bash
# Create root app (app of apps)
kubectl apply -f applications/root.yaml

# ArgoCD will automatically create and sync all child apps
```

## SOPS Integration

### Setup SOPS with ArgoCD

```bash
# 1. Generate AGE key (if you don't have one)
age-keygen -o ~/.config/sops/age/keys.txt

# 2. Save your public key from the output
# Example: age14c4u5y4uf869su746ghhear0ys4p6fuj6r0pvzrujj03dwh3xals40k30e

# 3. Update .sops.yaml with your public key
# Edit: ArgoCD/.sops.yaml and replace the age public key

# 4. Create secret in ArgoCD namespace (must be done manually!)
kubectl create secret generic sops-age-key \
  -n argocd \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt

# 5. Verify the secret was created
kubectl get secret sops-age-key -n argocd

# IMPORTANT: The base/sops-age-secret.yaml file is a TEMPLATE only!
# DO NOT uncomment or apply it - create the secret manually as shown above.
```

### Use SOPS-Encrypted Secrets

The `.sops.yaml` file is configured to encrypt specific fields automatically:

```bash
# Encrypt a Kubernetes secret (encrypts data/stringData fields only)
sops -e -i ../monitoring/base/grafana/secret.yaml

# Or encrypt the entire file
sops -e -i secret.yaml

# Edit encrypted file
sops ../monitoring/base/grafana/secret.yaml

# Commit to Git (only encrypted version is committed)
git add ../monitoring/base/grafana/secret.yaml
git commit -m "Add encrypted Grafana secret"
git push

# ArgoCD will automatically decrypt during sync using the AGE key
```

### SOPS Configuration

The `.sops.yaml` file defines encryption rules:

```yaml
creation_rules:
  - path_regex: .*\.(yaml|yml|json|env|ini)$
    age: >-
      YOUR_AGE_PUBLIC_KEY_HERE
    encrypted_regex: '^(data|stringData|password|token|key|secret)$'
```

This ensures only sensitive fields are encrypted, keeping the rest readable in git.

## Gateway API (Envoy Gateway)

This setup uses **Gateway API with Envoy Gateway** instead of the deprecated NGINX Ingress Controller.

### Architecture

- **GatewayClass**: Defines Envoy Gateway as the controller
- **Gateway**: Gets a LoadBalancer IP from MetalLB
- **HTTPRoute**: Routes traffic to ArgoCD server (replaces Ingress)

### Verify Gateway Setup

```bash
# Check Gateway has an external IP
kubectl get gateway -n envoy-gateway-system

# Check HTTPRoute is attached
kubectl get httproute -n argocd

# Check the Gateway service
kubectl get svc -n envoy-gateway-system
```

### Configure Hostname

Edit `base/httproute.yaml` to change the hostname:

```yaml
spec:
  hostnames:
    - 'argo.k8s.home' # Change this
```

### TLS with cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
            - name: eg
              namespace: envoy-gateway-system
EOF
```

## Projects & RBAC

### Homelab Project

The homelab project is included by default and all applications use it:

- Defined in: `examples/project-homelab.yaml`
- Used by: monitoring, gitlab, homepage, root app
- Includes RBAC roles: admin, readonly
- Configures allowed repositories and namespaces
- Enables orphaned resource warnings

```bash
# The project is already included in base/kustomization.yaml
# All applications are configured to use project: homelab

# View project details
kubectl get appproject homelab -n argocd -o yaml
```

### RBAC Roles

The homelab project defines two roles:

1. **Admin Role** (`homelab-admins` group)

   - Full access to applications, repositories, and clusters
   - Can sync, delete, and manage all resources

2. **Read-only Role** (`homelab-viewers` group)
   - View applications only
   - Cannot sync or modify resources

### Configure Users

See ArgoCD documentation for SSO/OIDC integration to map users to these groups.

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
    argocd.argoproj.io/sync-wave: '1' # Deploy first
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
      - /spec/replicas # Ignore HPA changes
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

### Quick Security Audit

```bash
# Verify SOPS secret exists (but don't view it!)
kubectl get secret sops-age-key -n argocd

# Check Gateway and HTTPRoute configuration
kubectl get gateway -n envoy-gateway-system
kubectl get httproute -n argocd -o yaml

# Verify applications use homelab project
kubectl get applications -n argocd -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.project}{"\n"}{end}'

# Check sync policies
kubectl get applications -n argocd -o yaml | grep -A 10 syncPolicy
```

## Documentation

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [SOPS Plugin](https://github.com/viaduct-ai/kustomize-sops)
- [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

ArgoCD will manage your entire cluster state via Git!
