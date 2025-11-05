# Bootstrap Files

This directory contains region-specific bootstrap ApplicationSets for initializing ArgoCD in each region.

## Files

- **eu-central-1.yaml** - Bootstrap for ArgoCD instance in Frankfurt (eu-central-1)
- **eu-south-1.yaml** - Bootstrap for ArgoCD instance in Milan (eu-south-1)

## Usage

### Deploy to eu-central-1 ArgoCD

```bash
# Connect to eu-central-1 ArgoCD
kubectl config use-context euc1-argocd

# Apply bootstrap
kubectl apply -f bootstrap/eu-central-1.yaml -n argocd
```

This will:
1. Scan `deployments/eu-central-1/` for cluster directories
2. Create ApplicationSets for discovered clusters: `euc1-dev-apps`, `euc1-staging-apps`, `euc1-prod-apps`
3. Each ApplicationSet scans deployment markers in its cluster directory
4. Applications are created only where deployment marker files exist

### Deploy to eu-south-1 ArgoCD

```bash
# Connect to eu-south-1 ArgoCD
kubectl config use-context eus1-argocd

# Apply bootstrap
kubectl apply -f bootstrap/eu-south-1.yaml -n argocd
```

This will:
1. Scan `deployments/eu-south-1/` for cluster directories
2. Create ApplicationSets for discovered clusters: `eus1-dev-apps`, `eus1-staging-apps`, `eus1-prod-apps`
3. Each ApplicationSet scans deployment markers in its cluster directory
4. Applications are created only where deployment marker files exist

## Verification

### Check ApplicationSets Created

```bash
kubectl get applicationsets -n argocd
```

Expected output (for eu-central-1):
```
NAME                 AGE
eu-central-1-root    1m
euc1-dev-apps        1m
euc1-staging-apps    1m
euc1-prod-apps       1m
```

### Check Applications Generated

```bash
kubectl get applications -n argocd
```

Expected output (for eu-central-1 based on deployment markers):
```
NAME                        SYNC STATUS   HEALTH STATUS
euc1-dev-frontend-app       Synced        Healthy
euc1-dev-backend-api        Synced        Healthy
euc1-dev-data-processor     Synced        Healthy
euc1-staging-frontend-app   Synced        Healthy
euc1-staging-backend-api    Synced        Healthy
euc1-prod-frontend-app      Synced        Healthy
```

Note: Applications are created ONLY for deployment markers that exist. In this example:
- `euc1-dev` has 3 apps (frontend-app, backend-api, data-processor)
- `euc1-staging` has 2 apps (frontend-app, backend-api)
- `euc1-prod` has 1 app (frontend-app)

## How It Works

### Bootstrap Flow

1. **Root ApplicationSet** (bootstrap file) scans `deployments/{region}/` for cluster directories
2. For each cluster directory found, it creates a **Cluster ApplicationSet**
3. Each Cluster ApplicationSet scans `deployments/{region}/{cluster}/*.yaml` for deployment markers
4. For each deployment marker + codebase combination, it creates an **Application** with:
   - Remote Helm chart from the codebase repository
   - Version from codebase.yaml matrix or deployment marker override
   - Values from `codebases/{codebase}/values/{region}/{environment}.yaml`
   - Cluster config from deployment marker file

### Cluster Discovery

The bootstrap uses a Git Directory Generator to discover clusters:

```yaml
generators:
  - git:
      directories:
        - path: deployments/eu-central-1/*
```

This scans for subdirectories under `deployments/eu-central-1/`. Each directory represents a cluster:
- `deployments/eu-central-1/euc1-dev/` → cluster `euc1-dev`
- `deployments/eu-central-1/euc1-staging/` → cluster `euc1-staging`
- `deployments/eu-central-1/euc1-prod/` → cluster `euc1-prod`

### Parameters Passed

The bootstrap passes these parameters to cluster ApplicationSets:

- `clusterName` - The cluster directory name (e.g., `euc1-dev`)
- `region` - The AWS region (e.g., `eu-central-1`)
- `repoURL` - This GitOps repository URL
- `revision` - Git branch/tag to use (default: `main`)

## Regional Independence

Each region's ArgoCD operates independently:

- **eu-central-1 ArgoCD** only manages clusters in `deployments/eu-central-1/`
- **eu-south-1 ArgoCD** only manages clusters in `deployments/eu-south-1/`
- Both use shared codebase definitions from `codebases/`
- Each uses region-specific values from `codebases/{codebase}/values/{region}/`
- Each region has independent deployment decisions (deployment markers)

## Adding New Cluster

To add a new cluster, simply create a new directory under deployments:

```bash
# Create new cluster directory
mkdir -p deployments/eu-central-1/euc1-qa

# Add deployment markers for apps you want to deploy
cat > deployments/eu-central-1/euc1-qa/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-qa-k8s.example.com
  namespace: argocd
  environment: qa

deployment: {}
EOF

git add deployments/eu-central-1/euc1-qa
git commit -m "Add QA cluster"
git push
```

The bootstrap will automatically:
1. Detect the new `euc1-qa` directory
2. Create ApplicationSet `euc1-qa-apps`
3. That ApplicationSet will create Application `euc1-qa-frontend-app`

## Selective Deployment

The key feature of this architecture is **selective deployment**:

- Codebases can be defined in `codebases/` without being deployed anywhere
- Applications are created ONLY when deployment marker files exist
- You control exactly which codebases deploy to which clusters

Example:
```bash
# Define a new codebase (doesn't create any Applications yet)
cat > codebases/payment-service/codebase.yaml <<EOF
codebase:
  name: payment-service
  repoURL: https://github.com/myorg/payment-service
  chartPath: deploy-templates
  namespace: backend
  targetRevisions:
    eu-central-1:
      development: main
      staging: v1.0.0
      production: v1.0.0
EOF

# Deploy to dev only
cat > deployments/eu-central-1/euc1-dev/payment-service.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development
deployment: {}
EOF

# Result: Application euc1-dev-payment-service is created
# No Applications created for staging or production (yet)
```

## Disaster Recovery

If one region fails:
1. The other region continues operating normally
2. GitOps repository remains the source of truth
3. Restore ArgoCD in the failed region
4. Re-apply the bootstrap file for that region
5. ArgoCD will recreate all ApplicationSets and Applications from Git

## Troubleshooting

### No ApplicationSets Created

**Check bootstrap ApplicationSet**:
```bash
kubectl describe applicationset eu-central-1-root -n argocd
```

**Common issues**:
- Repository URL not accessible from ArgoCD
- No cluster directories exist under `deployments/eu-central-1/`
- RBAC permissions missing

### Cluster ApplicationSets Not Created

**Check if cluster directories exist**:
```bash
ls deployments/eu-central-1/
```

Expected: `euc1-dev`, `euc1-staging`, `euc1-prod`

**Force bootstrap to refresh**:
```bash
kubectl patch applicationset eu-central-1-root -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'
```

### Applications Not Generated

**Check cluster ApplicationSet**:
```bash
kubectl describe applicationset euc1-dev-apps -n argocd
```

**Common issues**:
- No deployment marker files in `deployments/eu-central-1/euc1-dev/`
- Codebase.yaml doesn't exist for the codebase referenced in deployment marker
- Values file missing: `codebases/{codebase}/values/eu-central-1/development.yaml`

### Check Deployment Markers

```bash
# List deployment markers for a cluster
ls deployments/eu-central-1/euc1-dev/

# View a deployment marker
cat deployments/eu-central-1/euc1-dev/frontend-app.yaml
```

### Force ApplicationSet Refresh

To force ApplicationSet to re-evaluate:

```bash
kubectl patch applicationset euc1-dev-apps -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'
```

## Configuration

Before deploying, update the `repoURL` in both bootstrap files to match your actual Git repository:

```yaml
repoURL: https://github.com/YourOrg/argocd-app-of-app.git
```

Update the Git revision if you want to use a different branch:

```yaml
revision: main  # or your desired branch/tag
```

## Next Steps

After bootstrapping:

1. **Verify ApplicationSets**: Check that cluster ApplicationSets are created
2. **Verify Applications**: Check that Applications are created for deployment markers
3. **Check Sync Status**: Ensure Applications are syncing successfully
4. **Monitor Health**: Check health of deployed resources

For adding new clusters or codebases, see:
- Main [README](../README.md) for common operations
- [Deployment Markers Guide](../deployments/README.md) for deployment marker format
- [Complete Architecture Design](../docs/cluster-oriented-deployment-design.md) for in-depth documentation

## Removing Bootstrap

To remove all managed resources:

```bash
# Delete the bootstrap ApplicationSet
kubectl delete applicationset eu-central-1-root -n argocd

# This will cascade delete:
# - All cluster ApplicationSets (euc1-dev-apps, etc.)
# - All Applications
# - All deployed resources (if cascade delete is enabled)
```

**Warning**: This will undeploy all applications managed by this bootstrap!
