# ArgoCD Apps-of-Apps Repository

This repository implements an apps-of-apps pattern using ArgoCD ApplicationSet to automate the deployment of applications across multiple clusters.

## Overview

This GitOps repository manages deployments by:
- **Automatically generating ArgoCD Applications** based on codebase and cluster combinations
- **Deploying from remote Helm charts** located in application repositories
- **Applying cluster-specific value overrides** from this repository
- **Enabling self-service onboarding** for new applications and clusters

## Quick Start

### 1. Deploy the Root ApplicationSet

Apply the bootstrap ApplicationSet to your ArgoCD instance:

```bash
kubectl apply -f bootstrap/root-appset.yaml -n argocd
```

This will:
- Scan the `clusters/` directory
- Create one ApplicationSet per cluster
- Each cluster ApplicationSet will scan `codebases/` and create Applications

### 2. Verify Deployment

Check that ApplicationSets are created:

```bash
kubectl get applicationsets -n argocd
```

Check that Applications are generated:

```bash
kubectl get applications -n argocd
```

You should see Applications like:
- `frontend-app-dev`
- `frontend-app-staging`
- `frontend-app-prod`
- `backend-api-dev`
- etc.

## Repository Structure

```
├── bootstrap/              # Entry point - apply this first
│   └── root-appset.yaml   # Root ApplicationSet
│
├── clusters/               # Cluster definitions
│   ├── dev/
│   ├── staging/
│   └── prod/
│
├── codebases/             # Application configurations
│   ├── frontend-app/
│   │   ├── codebase.yaml # Metadata (repo URL, chart path)
│   │   └── values/       # Cluster-specific overrides
│   ├── backend-api/
│   └── data-processor/
│
├── applicationsets/       # ApplicationSet templates
│   └── cluster-apps.yaml # Generates Applications
│
└── docs/                  # Documentation
    ├── onboarding.md
    └── architecture.md
```

## How It Works

### Application Generation Flow

```
Root ApplicationSet
    ↓ (scans clusters/)
Per-Cluster ApplicationSets
    ↓ (scans codebases/)
Applications (codebase × cluster matrix)
```

### Example Generated Application

For codebase `frontend-app` and cluster `dev`, an Application is created:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app-dev
spec:
  sources:
    # Source 1: Remote Helm chart
    - repoURL: https://github.com/myorg/frontend-app
      path: deploy-templates
      helm:
        valueFiles:
          - values.yaml  # From remote chart
          - $values/codebases/frontend-app/values/dev.yaml

    # Source 2: Local values overrides
    - repoURL: https://github.com/YourOrg/argocd-app-of-app.git
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
```

## Adding a New Application (Codebase)

1. Create a directory under `codebases/`:

```bash
mkdir -p codebases/my-new-app/values
```

2. Create `codebase.yaml`:

```yaml
codebase:
  name: my-new-app
  repoURL: https://github.com/myorg/my-new-app
  chartPath: deploy-templates
  targetRevision: main
  namespace: my-app
```

3. Create values overrides for each cluster:

```bash
# Create values files
touch codebases/my-new-app/values/dev.yaml
touch codebases/my-new-app/values/staging.yaml
touch codebases/my-new-app/values/prod.yaml
```

4. Add your overrides (see examples in existing codebases)

5. Commit and push:

```bash
git add codebases/my-new-app/
git commit -m "Add my-new-app codebase"
git push
```

ArgoCD will automatically detect the changes and create Applications for all clusters!

## Adding a New Cluster

1. Create a directory under `clusters/`:

```bash
mkdir -p clusters/qa
```

2. Create `config.yaml`:

```yaml
cluster:
  name: qa
  server: https://qa-k8s.example.com
  namespace: argocd
  environment: qa
```

3. Add corresponding values files for each codebase:

```bash
touch codebases/frontend-app/values/qa.yaml
touch codebases/backend-api/values/qa.yaml
touch codebases/data-processor/values/qa.yaml
```

4. Commit and push:

```bash
git add clusters/qa codebases/*/values/qa.yaml
git commit -m "Add QA cluster"
git push
```

ArgoCD will create a new ApplicationSet for the QA cluster and deploy all codebases!

## Configuration

### Updating the Repository URL

Before deploying, update the repository URL in:
- `bootstrap/root-appset.yaml` - Change `repoURL` to your repository
- Update the example URLs in this README

### Cluster Server URLs

Update cluster server URLs in `clusters/*/config.yaml` to match your actual Kubernetes clusters.

### Sync Policies

All ApplicationSets are configured with automated sync:
- `prune: true` - Remove resources when removed from Git
- `selfHeal: true` - Auto-sync when resources drift

Modify `applicationsets/cluster-apps.yaml` to change sync behavior.

## Troubleshooting

### ApplicationSet Not Creating Applications

Check ApplicationSet status:

```bash
kubectl describe applicationset <name> -n argocd
```

### Application Not Syncing

Check Application status:

```bash
kubectl get application <name> -n argocd
argocd app get <name>
```

### Values Override Not Applied

Ensure:
1. Values file exists at `codebases/<codebase>/values/<cluster>.yaml`
2. File is committed to Git
3. ArgoCD has synced the ApplicationSet

## Advanced Topics

### Using Secrets

For sensitive values:
- Use ArgoCD Vault Plugin
- Use External Secrets Operator
- Use Sealed Secrets
- Reference secrets from values files

Example in values override:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

### Multiple Sources

The ApplicationSet uses multiple sources:
- Remote Helm chart (primary)
- Local values repository (overrides)

This allows keeping deployment templates in application repos while centralizing environment-specific configs.

### Filtering Deployments

To deploy only certain codebases to certain clusters, modify the ApplicationSet generators to add filters based on labels or annotations.

## Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/)
- [Akuity GitOps Workshops](https://docs.akuity.io/tutorials/)
- [Design Documentation](./DESIGN.md)
- [Onboarding Guide](./docs/onboarding.md)

## Support

For issues or questions:
1. Check the [troubleshooting section](#troubleshooting)
2. Review the [documentation](./docs/)
3. Open an issue in this repository
