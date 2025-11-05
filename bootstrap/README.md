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
1. Scan `regions/eu-central-1/clusters/*` for cluster definitions
2. Create ApplicationSets: `euc1-dev-apps`, `euc1-staging-apps`, `euc1-prod-apps`
3. Each ApplicationSet generates Applications for all codebases

### Deploy to eu-south-1 ArgoCD

```bash
# Connect to eu-south-1 ArgoCD
kubectl config use-context eus1-argocd

# Apply bootstrap
kubectl apply -f bootstrap/eu-south-1.yaml -n argocd
```

This will:
1. Scan `regions/eu-south-1/clusters/*` for cluster definitions
2. Create ApplicationSets: `eus1-dev-apps`, `eus1-staging-apps`, `eus1-prod-apps`
3. Each ApplicationSet generates Applications for all codebases

## Verification

### Check ApplicationSets Created

```bash
kubectl get applicationsets -n argocd
```

Expected output (for eu-central-1):
```
NAME              AGE
euc1-dev-apps     1m
euc1-staging-apps 1m
euc1-prod-apps    1m
```

### Check Applications Generated

```bash
kubectl get applications -n argocd
```

Expected output (for eu-central-1 with 3 codebases):
```
NAME                        SYNC STATUS   HEALTH STATUS
euc1-dev-frontend-app       Synced        Healthy
euc1-dev-backend-api        Synced        Healthy
euc1-dev-data-processor     Synced        Healthy
euc1-staging-frontend-app   Synced        Healthy
euc1-staging-backend-api    Synced        Healthy
euc1-staging-data-processor Synced        Healthy
euc1-prod-frontend-app      Synced        Healthy
euc1-prod-backend-api       Synced        Healthy
euc1-prod-data-processor    Synced        Healthy
```

## How It Works

### Bootstrap Flow

1. **Root ApplicationSet** (this file) scans the regional cluster directories
2. For each cluster found, it creates a **Cluster ApplicationSet**
3. Each Cluster ApplicationSet scans all codebases
4. For each codebase, it creates an **Application** with:
   - Remote Helm chart from the codebase repository
   - Values from `codebases/*/values/{region}/{environment}.yaml`

### Parameters Passed

The bootstrap passes these parameters to cluster ApplicationSets:

- `clusterName` - The cluster directory name (e.g., `euc1-dev`)
- `region` - The AWS region (e.g., `eu-central-1`)
- `repoURL` - This GitOps repository URL
- `revision` - Git branch/tag to use (default: `main`)

## Regional Independence

Each region's ArgoCD operates independently:

- **eu-central-1 ArgoCD** only manages clusters in `regions/eu-central-1/`
- **eu-south-1 ArgoCD** only manages clusters in `regions/eu-south-1/`
- Both use shared codebase definitions from `codebases/`
- Each uses region-specific values from `codebases/*/values/{region}/`

## Disaster Recovery

If one region fails:
1. The other region continues operating normally
2. GitOps repository remains the source of truth
3. Restore ArgoCD in the failed region
4. Re-apply the bootstrap file for that region
5. ArgoCD will recreate all Applications from Git

## Troubleshooting

### ApplicationSets Not Created

**Check**:
```bash
kubectl describe applicationset eu-central-1-root -n argocd
```

**Common issues**:
- Repository URL not accessible
- Cluster directory path incorrect
- RBAC permissions missing

### Applications Not Generated

**Check**:
```bash
kubectl describe applicationset euc1-dev-apps -n argocd
```

**Common issues**:
- Codebase files not found
- Values files missing for the region
- Matrix generator not finding matches

### Force Refresh

To force ApplicationSet to re-evaluate:

```bash
kubectl patch applicationset eu-central-1-root -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'
```

## Configuration

Before deploying, update the `repoURL` in both bootstrap files to match your actual Git repository:

```yaml
repoURL: https://github.com/YourOrg/argocd-app-of-app.git
```

## Next Steps

After bootstrapping:
1. Verify all ApplicationSets are created
2. Verify all Applications are created
3. Check sync status of Applications
4. Monitor health of deployed resources

For adding new clusters or codebases, see the main [onboarding documentation](../docs/onboarding.md).
