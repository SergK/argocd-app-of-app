# ArgoCD Apps-of-Apps Repository

This repository implements an apps-of-apps pattern using ArgoCD ApplicationSet to automate the deployment of applications across multiple clusters and regions.

## Overview

This GitOps repository manages deployments across **two AWS regions** (eu-central-1 and eu-south-1) with independent ArgoCD instances:

- **Multi-region support** with independent ArgoCD instances per region
- **Automatically generating ArgoCD Applications** based on codebase and cluster combinations
- **Deploying from remote Helm charts** located in application repositories
- **Regional and cluster-specific value overrides** from this repository
- **Enabling self-service onboarding** for new applications, clusters, and regions

## Quick Start

### 1. Deploy Bootstrap to Each Region

Apply the region-specific bootstrap ApplicationSet to each ArgoCD instance:

**For eu-central-1 (Frankfurt) ArgoCD:**

```bash
# Connect to eu-central-1 ArgoCD
kubectl config use-context euc1-argocd

# Apply bootstrap
kubectl apply -f bootstrap/eu-central-1.yaml -n argocd
```

**For eu-south-1 (Milan) ArgoCD:**

```bash
# Connect to eu-south-1 ArgoCD
kubectl config use-context eus1-argocd

# Apply bootstrap
kubectl apply -f bootstrap/eu-south-1.yaml -n argocd
```

Each bootstrap will:
- Scan its region's clusters in `regions/{region}/clusters/`
- Create one ApplicationSet per cluster
- Each cluster ApplicationSet will scan `codebases/` and create Applications

### 2. Verify Deployment

Check that ApplicationSets are created:

```bash
kubectl get applicationsets -n argocd
```

Expected output (for eu-central-1):
```
NAME                  AGE
euc1-dev-apps         1m
euc1-staging-apps     1m
euc1-prod-apps        1m
```

Check that Applications are generated:

```bash
kubectl get applications -n argocd
```

You should see Applications like:
- `euc1-dev-frontend-app`
- `euc1-staging-frontend-app`
- `euc1-prod-frontend-app`
- `euc1-dev-backend-api`
- etc.

## Repository Structure

```text
├── bootstrap/                    # Entry point - apply region-specific bootstrap
│   ├── eu-central-1.yaml        # Bootstrap for Frankfurt ArgoCD
│   ├── eu-south-1.yaml          # Bootstrap for Milan ArgoCD
│   └── README.md
│
├── regions/                      # Regional cluster definitions
│   ├── eu-central-1/
│   │   └── clusters/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   └── eu-south-1/
│       └── clusters/
│           ├── dev/
│           ├── staging/
│           └── prod/
│
├── codebases/                    # Application configurations (shared across regions)
│   ├── frontend-app/
│   │   ├── codebase.yaml        # Metadata (repo URL, chart path)
│   │   └── values/              # Region and cluster-specific overrides
│   │       ├── eu-central-1/
│   │       │   ├── dev.yaml
│   │       │   ├── staging.yaml
│   │       │   └── prod.yaml
│   │       └── eu-south-1/
│   │           ├── dev.yaml
│   │           ├── staging.yaml
│   │           └── prod.yaml
│   ├── backend-api/
│   └── data-processor/
│
├── applicationsets/              # ApplicationSet templates
│   └── cluster-apps.yaml        # Generates Applications (used by both regions)
│
└── docs/                         # Documentation
    ├── architecture.md
    ├── onboarding.md
    └── multi-region-design.md
```

## How It Works

### Application Generation Flow

```text
Region-Specific Bootstrap (per ArgoCD instance)
    ↓ (scans regions/{region}/clusters/)
Per-Cluster ApplicationSets
    ↓ (scans codebases/)
Applications (codebase × cluster matrix)
```

**Example for eu-central-1:**
- Bootstrap scans `regions/eu-central-1/clusters/*`
- Creates: `euc1-dev-apps`, `euc1-staging-apps`, `euc1-prod-apps`
- Each scans all codebases and creates Applications

### Multi-Region Architecture

This repository supports **two independent ArgoCD instances**:

| Region | ArgoCD Location | Manages Clusters | Application Prefix |
|--------|----------------|------------------|-------------------|
| eu-central-1 | Frankfurt | euc1-dev, euc1-staging, euc1-prod | euc1-* |
| eu-south-1 | Milan | eus1-dev, eus1-staging, eus1-prod | eus1-* |

**Key Features:**
- Each ArgoCD operates independently
- Shared codebase definitions (no duplication)
- Regional values: `codebases/*/values/{region}/{environment}.yaml`
- Regional isolation for disaster recovery

### Example Generated Application

For codebase `frontend-app` in `eu-central-1` region, `dev` environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: euc1-dev-frontend-app
  labels:
    codebase: frontend-app
    cluster: euc1-dev
    region: eu-central-1
    environment: development
spec:
  sources:
    # Source 1: Remote Helm chart
    - repoURL: https://github.com/myorg/frontend-app
      path: deploy-templates
      helm:
        # Use short name for Helm release (not the full Application name)
        releaseName: frontend-app
        valueFiles:
          - values.yaml  # From remote chart
          - $values/codebases/frontend-app/values/eu-central-1/development.yaml

    # Source 2: Local values overrides
    - repoURL: https://github.com/YourOrg/argocd-app-of-app.git
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
```

### Naming Conventions

**Application Names**: Applications are named using the format `<cluster>-<codebase>` where cluster includes region prefix:
- Format: `{region-prefix}-{environment}-{codebase}`
- Examples: `euc1-dev-frontend-app`, `eus1-prod-backend-api`
- Benefits: Easy grouping by cluster, unique across all ArgoCD instances

**Cluster Names**: Clusters are prefixed with region identifier:
- eu-central-1: `euc1-dev`, `euc1-staging`, `euc1-prod`
- eu-south-1: `eus1-dev`, `eus1-staging`, `eus1-prod`

**Helm Release Names**: The Helm `releaseName` is set to just the codebase name (e.g., `frontend-app`). This keeps Helm release names clean and consistent across all clusters and regions, avoiding long names like `euc1-dev-frontend-app`.

Benefits:

- Consistent Helm release names across environments
- Easier to reference in Helm commands: `helm list -n frontend` shows `frontend-app` in all clusters
- Resource names remain the same regardless of cluster (e.g., Deployment is always `frontend-app`, not `dev-frontend-app`)

## Adding a New Application (Codebase)

1. Create a directory under `codebases/`:

```bash
mkdir -p codebases/my-new-app/values/{eu-central-1,eu-south-1}
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

3. Create values overrides for **each region and environment**:

```bash
# Create values files for eu-central-1
touch codebases/my-new-app/values/eu-central-1/development.yaml
touch codebases/my-new-app/values/eu-central-1/staging.yaml
touch codebases/my-new-app/values/eu-central-1/production.yaml

# Create values files for eu-south-1
touch codebases/my-new-app/values/eu-south-1/development.yaml
touch codebases/my-new-app/values/eu-south-1/staging.yaml
touch codebases/my-new-app/values/eu-south-1/production.yaml
```

4. Add your overrides (see examples in existing codebases)

5. Commit and push:

```bash
git add codebases/my-new-app/
git commit -m "Add my-new-app codebase"
git push
```

Both ArgoCD instances will automatically detect the changes and create Applications for all clusters in their respective regions!

## Adding a New Cluster to a Region

To add a new cluster (e.g., `qa`) to eu-central-1:

1. Create a directory under the region:

```bash
mkdir -p regions/eu-central-1/clusters/qa
```

2. Create `config.yaml` with region-prefixed name:

```yaml
cluster:
  # Cluster name with region prefix
  name: euc1-qa
  # AWS region
  region: eu-central-1
  # Environment type
  environment: qa
  # Kubernetes API server URL
  server: https://qa-k8s.euc1.example.com
  namespace: argocd
```

3. Add corresponding values files for each codebase:

```bash
touch codebases/frontend-app/values/eu-central-1/qa.yaml
touch codebases/backend-api/values/eu-central-1/qa.yaml
touch codebases/data-processor/values/eu-central-1/qa.yaml
```

4. Commit and push:

```bash
git add regions/eu-central-1/clusters/qa codebases/*/values/eu-central-1/qa.yaml
git commit -m "Add QA cluster to eu-central-1"
git push
```

The eu-central-1 ArgoCD will automatically create a new ApplicationSet (`euc1-qa-apps`) and deploy all codebases!

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
