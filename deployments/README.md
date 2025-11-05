# Deployment Markers

This directory contains deployment marker files that control which codebases are deployed to which clusters.

## Structure

```
deployments/
├── {region}/
│   └── {cluster}/
│       └── {codebase}.yaml
```

- **{region}**: AWS region (e.g., `eu-central-1`, `eu-south-1`)
- **{cluster}**: Cluster name (e.g., `euc1-dev`, `euc1-staging`, `euc1-prod`)
- **{codebase}**: Codebase/application name (e.g., `frontend-app`, `backend-api`)

## How It Works

**Creating a file enables deployment**: When you create a file at `deployments/{region}/{cluster}/{codebase}.yaml`, ArgoCD will:
1. Detect the deployment marker file
2. Load codebase configuration from `codebases/{codebase}/codebase.yaml`
3. Create an Application named `{cluster}-{codebase}`
4. Deploy the specified version to the cluster

**Deleting a file disables deployment**: Removing the file will cause ArgoCD to delete the Application and undeploy the service.

## File Format

### Standard Deployment

```yaml
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
```

### With Version Override

```yaml
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  # Override targetRevision for this cluster only
  targetRevision: hotfix/security-patch-v1.2.1
```

## Current Deployments

### EU-Central-1 Region

**euc1-dev** (Development):
- frontend-app
- backend-api
- data-processor

**euc1-staging** (Staging):
- frontend-app
- backend-api

**euc1-prod** (Production):
- frontend-app

### EU-South-1 Region

**eus1-dev** (Development):
- backend-api
- data-processor

**eus1-staging** (Staging):
- (none)

**eus1-prod** (Production):
- (none)

## Common Operations

### Deploy New Application to Development

```bash
# Create deployment marker
cat > deployments/eu-central-1/euc1-dev/new-service.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
EOF

git add deployments/eu-central-1/euc1-dev/new-service.yaml
git commit -m "Deploy new-service to euc1-dev"
git push
```

### Promote to Staging

```bash
# Create staging deployment marker
cat > deployments/eu-central-1/euc1-staging/new-service.yaml <<EOF
cluster:
  server: https://euc1-staging-k8s.example.com
  namespace: argocd
  environment: staging

deployment: {}
EOF

git add deployments/eu-central-1/euc1-staging/new-service.yaml
git commit -m "Promote new-service to euc1-staging"
git push
```

### Emergency Hotfix

```bash
# Override version for production
cat > deployments/eu-central-1/euc1-prod/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  targetRevision: hotfix/security-fix-v1.2.1
EOF

git add deployments/eu-central-1/euc1-prod/frontend-app.yaml
git commit -m "HOTFIX: Deploy security patch to production"
git push
```

### Disable Deployment

```bash
# Remove deployment marker to undeploy
git rm deployments/eu-central-1/euc1-staging/old-service.yaml
git commit -m "Disable old-service in staging"
git push
```

## Version Resolution

The `targetRevision` (Helm chart version/Git ref) is resolved with this priority:

1. **Deployment Override** (highest priority)
   - `deployment.targetRevision` in the deployment marker file
   - Use for emergency hotfixes or feature branch testing

2. **Codebase Matrix** (default)
   - `codebase.targetRevisions[region][environment]` from `codebases/{codebase}/codebase.yaml`
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

If no override exists in the deployment file, the ApplicationSet will use:
- `main` for euc1-dev
- `v1.3.0` for euc1-staging
- `v1.2.0` for euc1-prod

## Best Practices

1. **Always test in dev first**: Create deployment marker in dev environment before promoting

2. **Progressive rollout**: Follow dev → staging → production progression

3. **Descriptive commit messages**: Clearly describe what you're deploying and why

4. **Remove overrides after fix**: When hotfix is merged and tagged, remove the `deployment.targetRevision` override

5. **Keep files minimal**: Only add `deployment` fields when you need to override something

6. **Cleanup old deployments**: Remove marker files for decommissioned applications

## Troubleshooting

### Application Not Created

- Verify file path: `deployments/{region}/{cluster}/{codebase}.yaml`
- Confirm codebase exists: `codebases/{codebase}/codebase.yaml`
- Check ArgoCD ApplicationSet controller logs

### Wrong Version Deployed

- Check for `targetRevision` override in deployment file
- Verify `environment` matches expected environment name
- Confirm `targetRevisions` matrix in codebase.yaml has entry for this region+environment

### Sync Failures

- Verify `cluster.server` URL is correct
- Check namespace exists or CreateNamespace is enabled
- Confirm values file exists: `codebases/{codebase}/values/{region}/{environment}.yaml`
