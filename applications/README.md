# ArgoCD Applications

Application manifests for the App of Apps pattern.

## Files

- `root.yaml` - Root application that manages all other apps
- `linkding.yaml` - Bookmark manager
- `monitoring.yaml` - Prometheus/Grafana stack
- `gitlab.yaml` - GitLab instance
- `homepage.yaml` - Dashboard homepage
- `velero-minio.yaml` - Velero backup storage (MinIO)

## Usage

Apply after ArgoCD is running:
```bash
kubectl apply -f applications/root.yaml
```

The root app will automatically sync all other applications.
