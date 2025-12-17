# ArgoCD Base

Core ArgoCD installation with KSOPS support for encrypted secrets.

## Files

- `kustomization.yaml` - Main kustomize config
- `namespace.yaml` - argocd namespace
- `install.yaml` - ArgoCD manifest reference
- `ingress.yaml` - Ingress for argo.k8s.home
- `project-homelab.yaml` - AppProject with RBAC (apply after install)
- `repo-server-ksops-patch.yaml` - KSOPS sidecar for SOPS decryption
- `sops-age-secret.yaml` - AGE key secret template
- `argocd-cm-patch.yaml` - ConfigMap customizations

## Install

```bash
kubectl apply -k base/
```
