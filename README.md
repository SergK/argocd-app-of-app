# ArgoCD Apps-of-Apps Repository

This repository implements a **cluster-oriented deployment** pattern using ArgoCD ApplicationSet to automate the deployment of applications across multiple clusters and regions.

## Overview

This GitOps repository manages deployments across **two AWS regions** (eu-central-1 and eu-south-1) with independent ArgoCD instances, featuring:

- **Selective deployment** - deploy codebases to specific clusters only (no forced all-or-nothing deployments)
- **Progressive rollout** - add deployment markers when ready to promote to higher environments
- **Multi-region support** - independent ArgoCD instances per region
- **File-based deployment control** - creating a file enables deployment
- **Per-cluster version control** - different Helm chart versions per cluster
- **Remote Helm charts** - deploy from application repositories
- **Values overrides** - region and environment-specific configurations

## Key Concept

**Creating a file enables deployment**. When you create a deployment marker file at `deployments/{region}/{cluster}/{codebase}.yaml`, ArgoCD will automatically:

1. Detect the new deployment marker
2. Load the codebase configuration
3. Create an Application
4. Deploy to the specified cluster

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
- Scan deployment directories in `deployments/{region}/`
- Create one ApplicationSet per cluster
- Each ApplicationSet scans deployment markers for that cluster
- Applications are created only where deployment markers exist

### 2. Verify Deployment

Check that cluster-level ApplicationSets are created:

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

## Repository Structure

```text
├── bootstrap/                              # Entry point - region-specific bootstraps
│   ├── eu-central-1.yaml                   # Bootstrap for Frankfurt ArgoCD
│   ├── eu-south-1.yaml                     # Bootstrap for Milan ArgoCD
│   └── README.md                           # Bootstrap documentation
│
├── deployments/                            # Deployment markers (file existence = deployment enabled)
│   ├── eu-central-1/
│   │   ├── euc1-dev/                       # Development cluster deployments
│   │   │   ├── frontend-app.yaml
│   │   │   ├── backend-api.yaml
│   │   │   └── data-processor.yaml
│   │   ├── euc1-staging/                   # Staging cluster deployments
│   │   │   ├── frontend-app.yaml
│   │   │   └── backend-api.yaml
│   │   └── euc1-prod/                      # Production cluster deployments
│   │       └── frontend-app.yaml
│   └── eu-south-1/
│       └── eus1-dev/
│           ├── backend-api.yaml
│           └── data-processor.yaml
│
├── codebases/                              # Application configurations (shared across regions)
│   ├── frontend-app/
│   │   ├── codebase.yaml                   # Metadata + version matrix
│   │   └── values/                         # Region and environment-specific overrides
│   │       ├── eu-central-1/
│   │       │   ├── development.yaml
│   │       │   ├── staging.yaml
│   │       │   └── production.yaml
│   │       └── eu-south-1/
│   │           ├── development.yaml
│   │           ├── staging.yaml
│   │           └── production.yaml
│   ├── backend-api/
│   └── data-processor/
│
├── applicationsets/
│   └── cluster-apps.yaml                   # ApplicationSet template (used by bootstrap)
│
└── docs/
    ├── cluster-oriented-deployment-design.md  # Complete architecture documentation
    └── onboarding.md                        # Onboarding guide

```

## How It Works

### Bootstrap Flow

```
Bootstrap ApplicationSet (per region)
    ↓ (scans deployments/{region}/ directories)
Cluster ApplicationSets (one per cluster)
    ↓ (scans deployments/{region}/{cluster}/*.yaml)
Applications (one per deployment marker + codebase combination)
    ↓ (deploys to target cluster)
Kubernetes Resources
```

### Example: Deploying New Application

**Step 1: Define the codebase** (once)

```bash
# Create codebase configuration
cat > codebases/payment-service/codebase.yaml <<EOF
codebase:
  name: payment-service
  repoURL: https://github.com/myorg/payment-service
  chartPath: deploy-templates
  namespace: backend

  # Version matrix: define versions for all regions and environments
  targetRevisions:
    eu-central-1:
      development: main
      staging: v1.0.0
      production: v1.0.0
    eu-south-1:
      development: main
      staging: v1.0.0
      production: v1.0.0
EOF

# Create environment-specific values
mkdir -p codebases/payment-service/values/eu-central-1
cat > codebases/payment-service/values/eu-central-1/development.yaml <<EOF
replicaCount: 1
resources:
  requests:
    cpu: 100m
    memory: 128Mi
EOF
```

**At this point**: No Applications are created. The codebase is defined but not deployed anywhere.

**Step 2: Deploy to development**

```bash
# Enable deployment by creating marker file
mkdir -p deployments/eu-central-1/euc1-dev

cat > deployments/eu-central-1/euc1-dev/payment-service.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
EOF

git add deployments/eu-central-1/euc1-dev/payment-service.yaml
git commit -m "Deploy payment-service to euc1-dev"
git push
```

**Result**: Application `euc1-dev-payment-service` is created and deploys from `main` branch.

**Step 3: Promote to staging** (after testing)

```bash
# Enable staging deployment
cat > deployments/eu-central-1/euc1-staging/payment-service.yaml <<EOF
cluster:
  server: https://euc1-staging-k8s.example.com
  namespace: argocd
  environment: staging

deployment: {}
EOF

git add deployments/eu-central-1/euc1-staging/payment-service.yaml
git commit -m "Promote payment-service to euc1-staging"
git push
```

**Result**: Application `euc1-staging-payment-service` is created and deploys from `v1.0.0` tag.

**Step 4: Promote to production** (after successful staging tests)

```bash
# Enable production deployment
cat > deployments/eu-central-1/euc1-prod/payment-service.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment: {}
EOF

git add deployments/eu-central-1/euc1-prod/payment-service.yaml
git commit -m "Promote payment-service to production"
git push
```

## Common Operations

### Add New Codebase

1. Create `codebases/{codebase}/codebase.yaml` with version matrix
2. Create values files: `codebases/{codebase}/values/{region}/{environment}.yaml`
3. No Applications are created yet (intentional - you control when to deploy)

### Deploy to Cluster

Create deployment marker file:

```bash
cat > deployments/eu-central-1/euc1-dev/my-app.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
EOF
```

### Emergency Hotfix

Override version for specific cluster:

```bash
cat > deployments/eu-central-1/euc1-prod/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  # Override to deploy hotfix branch
  targetRevision: hotfix/security-patch-v1.2.1
EOF
```

### Disable Deployment

Remove the deployment marker file:

```bash
git rm deployments/eu-central-1/euc1-staging/old-service.yaml
git commit -m "Disable old-service in staging"
git push
```

### Add New Cluster

1. Create deployment directory: `deployments/{region}/{cluster}/`
2. Create deployment markers for applications you want to deploy
3. Bootstrap will automatically detect the new cluster directory

Example:

```bash
# Create QA cluster directory
mkdir -p deployments/eu-central-1/euc1-qa

# Deploy selected applications to QA
cat > deployments/eu-central-1/euc1-qa/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-qa-k8s.example.com
  namespace: argocd
  environment: qa

deployment: {}
EOF

git add deployments/eu-central-1/euc1-qa
git commit -m "Add QA cluster with frontend-app"
git push
```

## Version Resolution

The `targetRevision` (Helm chart version/Git ref) is resolved with this priority:

1. **Deployment Override** (highest priority)
   - `deployment.targetRevision` in deployment marker file
   - Use for emergency hotfixes or feature branch testing

2. **Codebase Matrix** (default)
   - `codebase.targetRevisions[region][environment]` from `codebase.yaml`
   - Standard version control per region and environment

Example:

```yaml
# codebases/frontend-app/codebase.yaml
targetRevisions:
  eu-central-1:
    development: main
    staging: v1.3.0
    production: v1.2.0
```

If no override exists in the deployment marker file:
- `euc1-dev` deploys from `main`
- `euc1-staging` deploys from `v1.3.0`
- `euc1-prod` deploys from `v1.2.0`

## Benefits

✅ **No forced deployments** - codebases can be defined without deploying to all clusters

✅ **Progressive rollout** - deploy to dev first, then staging, then production when ready

✅ **Independent cluster/codebase addition** - add either without affecting the other

✅ **File-based deployment control** - intuitive GitOps workflow

✅ **Emergency hotfix support** - override version for specific cluster only

✅ **Clear audit trail** - Git history shows when each deployment was enabled/disabled

✅ **Multi-region support** - each region has independent deployment decisions

✅ **Native ArgoCD** - uses only Matrix Generator and Git Generators, no custom plugins

## Documentation

- **[Complete Architecture Design](docs/cluster-oriented-deployment-design.md)** - In-depth design document with workflows, examples, and best practices
- **[Deployment Markers Guide](deployments/README.md)** - Quick reference for deployment marker files
- **[Bootstrap Guide](bootstrap/README.md)** - How bootstrap ApplicationSets work
- **[Onboarding Guide](docs/onboarding.md)** - How to onboard new applications and clusters

## Troubleshooting

### Application Not Created

- Verify file path: `deployments/{region}/{cluster}/{codebase}.yaml`
- Confirm codebase exists: `codebases/{codebase}/codebase.yaml`
- Check ArgoCD ApplicationSet controller logs

### Wrong Version Deployed

- Check for `targetRevision` override in deployment marker file
- Verify `environment` in deployment marker matches environment name
- Confirm `targetRevisions` matrix in codebase.yaml has entry for this region+environment

### Sync Failures

- Verify `cluster.server` URL is correct and reachable
- Check namespace exists or CreateNamespace is enabled
- Confirm values file exists: `codebases/{codebase}/values/{region}/{environment}.yaml`

## Migration from Previous Architecture

If you're migrating from the regions-based architecture:

1. Bootstrap files now scan `deployments/{region}/` instead of `regions/{region}/clusters/`
2. Create deployment marker files for currently deployed applications
3. Cluster configuration is now embedded in deployment marker files
4. The `regions/` directory is no longer used and can be removed

## Contributing

When adding new deployments:

1. **Test in dev first** - always create deployment marker in dev environment first
2. **Progressive rollout** - follow dev → staging → production progression
3. **Descriptive commit messages** - clearly describe what you're deploying and why
4. **PR reviews** - require reviews for production deployment marker changes
5. **Remove overrides** - when hotfix is merged and tagged, remove `targetRevision` override

## License

[Your License]
