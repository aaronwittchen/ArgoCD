# Overlays

Environment-specific configurations.

## Directories

- `local-path/` - For local-path storage provisioner
- `longhorn/` - For Longhorn distributed storage (recommended)

## Usage

```bash
# Deploy with Longhorn storage
kubectl apply -k overlays/longhorn/
```
