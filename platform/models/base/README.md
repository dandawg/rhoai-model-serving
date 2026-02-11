# Base Resources for Model Serving

This directory contains shared resources that are used by all model download jobs and serving instances.

## Resources

- **namespace.yaml**: Creates the `demo` namespace with proper labels and annotations
- **serviceaccount.yaml**: ServiceAccount `model-downloader` used by model download jobs
- **role.yaml**: Role and RoleBinding with permissions to:
  - Get/list pods and pod logs
  - Get/list/create PVCs
  - Get secrets

## GitOps Deployment

These base resources should be deployed **before** any individual model resources. The sync-wave annotations ensure proper ordering:

- Namespace: sync-wave `-2`
- ServiceAccount, Role, RoleBinding: sync-wave `-1`
- Model-specific resources: sync-wave `0` (default)

## ArgoCD Application

Deploy these resources using the ArgoCD Application:

```yaml
gitops/platform/models/00-base-resources.yaml
```

Or include it in the "all-in-one" applications like:
- `gitops/platform/qwen3-vl-4b-all.yaml`
- `gitops/platform/qwen3-vl-embedding-2b-all.yaml`

## Important

Individual model kustomizations should **NOT** reference these base resources directly. They are deployed separately to avoid GitOps conflicts when multiple models are deployed simultaneously.
