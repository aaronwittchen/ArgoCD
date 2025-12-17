# ArgoCD Deployment Guide

Complete guide for deploying ArgoCD for GitOps on Kubernetes.

## Quick Start

### Install ArgoCD

```bash
# Deploy ArgoCD
kubectl apply -k overlays/longhorn

# or

kubectl apply -k overlays/local-path

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### Access ArgoCD

#### Option 1: Port-Forward

```bash
# Port-forward the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Access: https://localhost:8080
# Username: admin
# Password: (from above)
```

#### Option 2: Ingress

```bash
# Access via Ingress like
echo "http://argocd.{IP}.nip.io" # or any other ingress url

# Get password (same as above)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Install ArgoCD CLI

```bash
# Arch Linux
sudo pacman -S argocd

# Or download binary
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login
argocd login localhost:8080
# Or
argocd login argocd.192.168.2.207.nip.io
```

## Deploy Your Applications

### Method 1: Via UI

if private repo
Add the repository in ArgoCD

Open ArgoCD UI → Settings → Repositories → Connect Repo.

Use these values:

Repo URL: https://github.com/aaronwittchen/Prometheus-Grafana-Setup.git

Username: your GitHub username

Password / Token: your PAT

Test connection → Save.

1. Click **+ New App**
2. Fill in details:
   - **Application Name**: monitoring-stack
   - **Project**: default
   - **Sync Policy**: Automatic
   - Enable Auto-Sync
     Prune Resources
     Self Heal
     Set Deletion Finalizer
     Sync Options
     Skip Schema Validation
     Auto-Create Namespace
     Prune Last
     Apply Out of Sync Only
     Respect Ignore Differences
     Server-Side Apply
     Prune Propagation Policy:
     foreground
     Replace
     Retry
   - **Repository URL**: https://github.com/YOUR_USERNAME/YOUR_REPO
   - **Path**: overlays/local-path
   - **Cluster URL**: https://kubernetes.default.svc
   - **Namespace**: monitoring
3. Click **Create**

### Method 2: Via CLI

```bash
# Deploy monitoring stack
argocd app create monitoring-stack \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path "Prometheus & Grafana Setup/overlays/local-path" \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --sync-policy automated \
  --sync-option CreateNamespace=true

# Deploy GitLab
argocd app create gitlab \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path "GitLab/overlays/local-path" \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace gitlab \
  --sync-policy automated \
  --sync-option CreateNamespace=true
```

### Method 3: Via Manifest (Recommended)

```bash
# Deploy individual apps
kubectl apply -f applications/monitoring.yaml
kubectl apply -f applications/gitlab.yaml

# OR deploy everything at once (App of Apps)
kubectl apply -f applications/root.yaml
```

## Configure SOPS Integration

### Step 1: Generate Age Key

```bash
# Install age
sudo pacman -S age

# Generate keypair
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt

# View public key (save this!)
grep 'public key:' ~/.config/sops/age/keys.txt
```

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install age -y
age --version
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
grep 'public key:' ~/.config/sops/age/keys.txt
```

### Step 2: Configure SOPS in Repository

```bash
# Create .sops.yaml in repository root
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*secret.*\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # Your public key
EOF

# Commit
git add .sops.yaml
git commit -m "Add SOPS configuration"
```

SOPS uses the public key to encrypt files. Only someone with the corresponding private key can decrypt them.

Committing the public key to your repo is normal and intended, so that anyone who has the private key can decrypt the secrets.

Your private key must never be committed.

### Step 3: Install SOPS Plugin in ArgoCD

```bash
# Create secret with age key
kubectl create secret generic sops-age-key \
  -n argocd \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt

# Apply SOPS plugin configuration
kubectl apply -f examples/sops-plugin.yaml

# Restart repo server
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

**Install SOPS CLI (if not already installed):**

```bash
# Linux
curl -L -o sops https://github.com/getsops/sops/releases/download/v3.11.0/sops-v3.11.0.linux.amd64
chmod +x sops
sudo mv sops /usr/local/bin/
sops --version
```

### Step 4: Encrypt Secrets

```bash
# Encrypt Grafana secret
cd "../Prometheus & Grafana Setup"
sops -e -i base/grafana/secret.yaml

# Encrypt GitLab secrets
cd "../GitLab"
sops -e -i base/secrets.yaml

# Commit encrypted secrets
git add -A
git commit -m "Add encrypted secrets"
git push
```

ArgoCD will now automatically decrypt secrets during sync!

## Configure Discord Notifications

### Step 1: Get Discord Webhook

1. Go to Discord Server Settings
2. Integrations → Webhooks
3. Create webhook
4. Copy URL

### Step 2: Configure ArgoCD Notifications

```bash
# Edit notifications config
kubectl edit configmap argocd-notifications-cm -n argocd

# Or apply example
# 1. Edit examples/notifications-discord.yaml
# 2. Replace DISCORD_WEBHOOK_URL placeholder with your actual webhook
kubectl apply -f examples/notifications-discord.yaml

# Create the notifications secret with your webhook URL
kubectl create secret generic argocd-notifications-secret \
   -n argocd \
   --from-literal=discord-webhook-url='https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN/slack'
# Restart notifications controller
kubectl rollout restart deployment/argocd-notifications-controller -n argocd
```

### Step 3: Test Notification

```bash
# Trigger a sync
argocd app sync monitoring-stack

# Check Discord for notification
```

## App of Apps Pattern

Deploy everything with one command:

```bash
# Create root app
kubectl apply -f applications/root.yaml

# ArgoCD will automatically:
# 1. Deploy monitoring stack (wave 1)
# 2. Deploy GitLab (wave 2)
# 3. Deploy any other apps (wave 3+)
```

## Management

### View Applications

```bash
# CLI
argocd app list

# Or in UI
# https://argocd.192.168.2.207.nip.io/applications
```

### Sync Application

```bash
# Sync specific app
argocd app sync monitoring-stack

# Sync all apps
argocd app sync --all

# Hard refresh (re-sync everything)
argocd app sync monitoring-stack --force --prune
```

### View Sync Status

```bash
# CLI
argocd app get monitoring-stack

# Watch sync progress
argocd app wait monitoring-stack --health
```

### View Application Details

```bash
# Get app details
argocd app get monitoring-stack

# View diff
argocd app diff monitoring-stack

# View logs
argocd app logs monitoring-stack

# View events
kubectl describe application monitoring-stack -n argocd
```

### Rollback Application

```bash
# View history
argocd app history monitoring-stack

# Rollback to previous version
argocd app rollback monitoring-stack
```

### Delete Application

```bash
# Delete app (keeps resources)
argocd app delete monitoring-stack --cascade=false

# Delete app and resources
argocd app delete monitoring-stack

# Force delete
argocd app delete monitoring-stack --cascade --force
```

## Best Practices

### 1. Use Projects

Organize apps by environment or team:

```bash
kubectl apply -f examples/project-homelab.yaml
```

### 2. Enable Auto-Sync

Let ArgoCD automatically sync changes:

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
    argocd.argoproj.io/sync-wave: '1'
```

### 4. Encrypt Secrets

Always use SOPS for secrets in Git:

```bash
sops -e -i secret.yaml
```

### 5. Monitor Sync Status

Set up Discord/Slack notifications for sync events.

## Useful Commands

```bash
# List all applications
argocd app list

# Get application details
argocd app get APP_NAME

# Sync application
argocd app sync APP_NAME

# Delete application
argocd app delete APP_NAME

# View application logs
argocd app logs APP_NAME

# View application history
argocd app history APP_NAME

# Rollback application
argocd app rollback APP_NAME

# Set auto-sync
argocd app set APP_NAME --sync-policy automated

# View cluster info
argocd cluster list

# Add repository
argocd repo add https://github.com/USER/REPO

# View repositories
argocd repo list

# Change password
argocd account update-password
```

## Next Steps

1. **Deploy monitoring**: `kubectl apply -f applications/monitoring.yaml`
2. **Deploy GitLab**: `kubectl apply -f applications/gitlab.yaml`
3. **Enable SOPS**: Follow SOPS integration steps above
4. **Configure notifications**: Set up Discord webhooks
5. **Create projects**: Organize your applications

ArgoCD is now managing your cluster via GitOps!
