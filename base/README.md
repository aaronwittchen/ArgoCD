# ArgoCD Base

Core ArgoCD installation with KSOPS support for encrypted secrets.

## Files

- `kustomization.yaml` - Main kustomize config
- `namespace.yaml` - argocd namespace
- `install.yaml` - ArgoCD manifest reference
- `gateway.yaml` - Gateway API GatewayClass and Gateway for Envoy
- `httproute.yaml` - HTTPRoute for argo.k8s.home (replaces Ingress)
- `project-homelab.yaml` - AppProject with RBAC (apply after install)
- `repo-server-ksops-patch.yaml` - KSOPS sidecar for SOPS decryption
- `sops-age-secret.yaml` - AGE key secret template
- `argocd-cm-patch.yaml` - ConfigMap customizations

## Gateway API

This setup uses Gateway API with Envoy Gateway instead of NGINX Ingress Controller.

**Prerequisites:**

- Envoy Gateway installed in the cluster
- MetalLB for LoadBalancer IP assignment

**Verify Gateway:**

```bash
kubectl get gateway -n envoy-gateway-system
kubectl get httproute -n argocd
```

## Install

```bash
kubectl apply -k base/
```
