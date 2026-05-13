# Helm Charts Repository - Agent Guidelines

## Image Tag Updates

When updating container image tags in any chart's `values.yaml`, you **must** also bump the chart version (patch version) in the corresponding `Chart.yaml`.

### Rules

1. **Always bump the chart version** when any image tag in `values.yaml` changes. Increment the patch version (e.g., `1.6.7` → `1.6.8`).
2. **Do not update charts** whose images did not change.
3. **Skip `auto-deploy-app`** — it uses a template placeholder image (`gitlab.example.com/group/project:stable`) and should never be updated.
4. **Verify** that the new image tag actually exists on the container registry before applying the change.

### Example

If `charts/servarr/values.yaml` has an image tag update:
- Edit `charts/servarr/values.yaml` with the new tag
- Edit `charts/servarr/Chart.yaml` and bump `version: X.Y.Z` → `version: X.Y.Z+1`