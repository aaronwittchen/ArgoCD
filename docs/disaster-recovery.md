# ArgoCD Disaster Recovery

This document outlines backup and recovery procedures for the ArgoCD installation.

## Prerequisites

- `kubectl` access to the cluster
- SOPS AGE private key backed up securely
- Access to the Git repository

## Backup Procedures

### 1. Backup Admin Credentials

```bash
# Export admin secret
kubectl get secret -n argocd argocd-initial-admin-secret -o yaml > backup/admin-secret.yaml

# Get current admin password (for reference)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. Backup All Applications

```bash
# Create backup directory
mkdir -p backup/$(date +%Y%m%d)

# Export all ArgoCD applications
kubectl get applications -n argocd -o yaml > backup/$(date +%Y%m%d)/applications.yaml

# Export all AppProjects
kubectl get appprojects -n argocd -o yaml > backup/$(date +%Y%m%d)/appprojects.yaml

# Export ApplicationSets (if used)
kubectl get applicationsets -n argocd -o yaml > backup/$(date +%Y%m%d)/applicationsets.yaml
```

### 3. Backup ArgoCD Configuration

```bash
# Export ConfigMaps
kubectl get configmap -n argocd argocd-cm -o yaml > backup/$(date +%Y%m%d)/argocd-cm.yaml
kubectl get configmap -n argocd argocd-rbac-cm -o yaml > backup/$(date +%Y%m%d)/argocd-rbac-cm.yaml

# Export Secrets (encrypted repos, clusters, etc.)
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository -o yaml > backup/$(date +%Y%m%d)/repo-secrets.yaml
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster -o yaml > backup/$(date +%Y%m%d)/cluster-secrets.yaml
```

### 4. Backup SOPS AGE Key

**CRITICAL**: Store the AGE private key securely outside the cluster.

```bash
# Location of AGE key (typically)
cat ~/.config/sops/age/keys.txt

# Store securely in:
# - Password manager (1Password, Bitwarden)
# - Hardware security module
# - Encrypted offline storage
```

### 5. Full Cluster Backup Script

```bash
#!/bin/bash
# backup-argocd.sh

BACKUP_DIR="backup/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Backing up ArgoCD to $BACKUP_DIR..."

# Namespace resources
kubectl get all -n argocd -o yaml > "$BACKUP_DIR/all-resources.yaml"

# Applications
kubectl get applications -n argocd -o yaml > "$BACKUP_DIR/applications.yaml"
kubectl get appprojects -n argocd -o yaml > "$BACKUP_DIR/appprojects.yaml"
kubectl get applicationsets -n argocd -o yaml > "$BACKUP_DIR/applicationsets.yaml"

# ConfigMaps
kubectl get configmap -n argocd -o yaml > "$BACKUP_DIR/configmaps.yaml"

# Secrets (be careful with these!)
kubectl get secret -n argocd -o yaml > "$BACKUP_DIR/secrets.yaml"

echo "Backup complete: $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

## Recovery Procedures

### Scenario 1: Full Cluster Recovery

1. **Recreate the cluster** (if needed)

2. **Restore SOPS AGE key**:
   ```bash
   mkdir -p ~/.config/sops/age
   # Restore keys.txt from secure backup
   ```

3. **Create ArgoCD namespace and SOPS secret**:
   ```bash
   kubectl create namespace argocd
   kubectl create secret generic sops-age-key -n argocd \
     --from-file=keys.txt=$HOME/.config/sops/age/keys.txt
   ```

4. **Apply ArgoCD installation**:
   ```bash
   kubectl apply -k overlays/longhorn
   # or
   kubectl apply -k overlays/local-path
   ```

5. **Wait for ArgoCD to be ready**:
   ```bash
   kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
   ```

6. **Restore applications** (if not using GitOps):
   ```bash
   kubectl apply -f backup/applications.yaml
   ```

7. **Apply root application** (GitOps approach - recommended):
   ```bash
   kubectl apply -f applications/root.yaml
   ```

### Scenario 2: ArgoCD Component Failure

1. **Check pod status**:
   ```bash
   kubectl get pods -n argocd
   kubectl describe pod <failing-pod> -n argocd
   ```

2. **Restart failing components**:
   ```bash
   kubectl rollout restart deployment/argocd-server -n argocd
   kubectl rollout restart deployment/argocd-repo-server -n argocd
   kubectl rollout restart statefulset/argocd-application-controller -n argocd
   ```

3. **Check logs**:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=100
   ```

### Scenario 3: SOPS Key Recovery

If the SOPS AGE key is lost:

1. **Generate new AGE key**:
   ```bash
   age-keygen -o ~/.config/sops/age/keys.txt
   ```

2. **Update .sops.yaml** with new public key

3. **Re-encrypt all secrets**:
   ```bash
   # Decrypt with old key (if available) and re-encrypt with new
   sops -d secret.yaml | sops -e > secret.yaml.new
   mv secret.yaml.new secret.yaml
   ```

4. **Update Kubernetes secret**:
   ```bash
   kubectl delete secret sops-age-key -n argocd
   kubectl create secret generic sops-age-key -n argocd \
     --from-file=keys.txt=$HOME/.config/sops/age/keys.txt
   ```

5. **Restart repo-server**:
   ```bash
   kubectl rollout restart deployment/argocd-repo-server -n argocd
   ```

### Scenario 4: Application Sync Issues

1. **Hard refresh application**:
   ```bash
   argocd app get <app-name> --hard-refresh
   ```

2. **Force sync**:
   ```bash
   argocd app sync <app-name> --force
   ```

3. **Delete and recreate application**:
   ```bash
   kubectl delete application <app-name> -n argocd
   kubectl apply -f applications/<app-name>.yaml
   ```

## Verification Commands

```bash
# Check ArgoCD health
kubectl get pods -n argocd
kubectl get applications -n argocd

# Check sync status
argocd app list

# Verify SOPS decryption
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server | grep -i sops

# Test application sync
argocd app sync <app-name> --dry-run
```

## Regular Maintenance

### Weekly
- Verify all applications are synced
- Check for pending updates
- Review ArgoCD logs for errors

### Monthly
- Test backup restoration procedure
- Rotate admin credentials if needed
- Update ArgoCD version (if available)

### Quarterly
- Full disaster recovery drill
- Review and update this documentation
- Audit RBAC permissions

## Important Files

| File | Purpose |
|------|---------|
| `~/.config/sops/age/keys.txt` | AGE private key (NEVER commit) |
| `.sops.yaml` | SOPS encryption configuration |
| `applications/root.yaml` | Root app that bootstraps all others |
| `overlays/longhorn/` | Production overlay with Longhorn storage |

## Contact

For emergencies, ensure you have:
- Access to the Git repository
- SOPS AGE private key
- Kubernetes cluster credentials
- This documentation
