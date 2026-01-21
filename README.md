# Flux D2 Fleet Repository

This repository manages the Flux configuration for multiple Kubernetes clusters using the Flux Operator and OCI artifacts.

## Architecture

- **Flux Operator**: Manages Flux installation via FluxInstance CRs
- **OCI Artifacts**: Cluster manifests stored in `ghcr.io/bendwyer/d2-fleet`
- **ResourceSets**: Template-based resource generation for components

## Clusters

- **omni**: Production cluster running Omni (`omni.bendwyer.cloud`)
- **home**: Production home cluster (`home.bendwyer.cloud`)

Both clusters sync from the `:latest` OCI tag. Changes merged to `main` are automatically published and reconciled.

## Repository Structure

```
flux-d2-fleet/
├── clusters/                           # Per-cluster configurations
│   ├── {cluster}/
│   │   ├── flux-system/
│   │   │   └── flux-instance.yaml      # Flux Operator bootstrap
│   │   └── tenants.yaml                # References tenants/{cluster} directory
│   └── ...
├── tenants/                            # Per-cluster ResourceSets
│   ├── {cluster}/
│   │   ├── apps.yaml                   # Application deployments
│   │   └── infra.yaml                  # Infrastructure deployments
│   └── ...
└── .github/workflows/                  # CI/CD automation
    ├── publish-oci.yml                 # OCI publishing with PR validation
    └── e2e.yml                         # E2E testing in KIND cluster
```

## Bootstrap Process

### Prerequisites

1. Flux Operator installed via Talos `extraManifests`:
   ```yaml
    ---
    cluster:
      extraManifests:
         - https://github.com/controlplaneio-fluxcd/flux-operator/releases/latest/download/install.yaml
   ```

2. 1Password bootstrap credentials added to Talos nodes:
   - `onepassword-credentials` Secret (1Password Connect credentials)
   - `onepassword-token` Secret (Operator token)

3. `kubectl` access to target cluster

### Bootstrap Commands

**Omni cluster:**
```bash
kubectl apply -f clusters/omni/flux-system/flux-instance.yaml
kubectl -n flux-system wait fluxinstance/flux --for=condition=Ready --timeout=5m
```

**Home cluster:**
```bash
kubectl apply -f clusters/home/flux-system/flux-instance.yaml
kubectl -n flux-system wait fluxinstance/flux --for=condition=Ready --timeout=5m
```

**Verify bootstrap:**
```bash
kubectl get fluxinstance -n flux-system
kubectl get ocirepository -n flux-system
kubectl get kustomization -n flux-system
```

## Ongoing Workflow

1. Edit manifests in this repo or component repos (flux-d2-infra, flux-d2-apps)
2. Create PR
3. GitHub Actions automatically validates:
   - **publish-oci**: Validates OCI artifact builds (doesn't publish on PRs)
   - **e2e**: Deploys to KIND cluster and tests Flux reconciliation
4. Review PR and merge to `main`
5. GitHub Actions on `main`:
   - Builds OCI artifacts
   - Only publishes if different from `:latest` (diff detection)
   - Signs with cosign
   - Pushes to GHCR
6. Both clusters reconcile within 1 minute (interval: 12h with retries every 3m)

## Adding Components

Edit `tenants/infra.yaml` or `tenants/apps.yaml` to add new component inputs:

```yaml
spec:
  inputs:
    - component: "cert-manager"
      tag: "latest"
      environment: "omni"
```

**Important:** Components must be added to the component repository's workflow matrix:
- Infrastructure: `flux-d2-infra/.github/workflows/publish-oci.yml`
- Applications: `flux-d2-apps/.github/workflows/publish-oci.yml`

## Secrets Management

Secrets are managed via **1Password Operator**:

1. **Bootstrap Secrets**: Injected via Terraform at cluster creation
   - `onepassword-credentials` (1Password Connect credentials)
   - `onepassword-token` (Operator authentication token)

2. **Application Secrets**: Managed via `OnePasswordItem` CRDs in component repos
   - Created in flux-d2-infra/configs or flux-d2-apps components
   - Automatically synced from 1Password vaults
   - Referenced in Deployments via postBuild substitution

## GitHub Actions Workflows

### publish-oci.yml
- **On PR**: Validates OCI artifact builds without publishing
- **On push to main**: Publishes artifacts to GHCR (only if changed)
- **Manual trigger**: Available via workflow_dispatch

### e2e.yml
- **On PR**: Deploys Flux to KIND cluster and validates ResourceSets
- **Manual trigger**: Available for troubleshooting
- **Tests**: YAML syntax, template rendering, Flux reconciliation

## References

- [Flux Operator Documentation](https://fluxoperator.dev/)
- [ControlPlane D2 Reference Architecture](https://github.com/controlplaneio-fluxcd/d2-fleet)
- [1Password Operator](https://github.com/1Password/onepassword-operator)
