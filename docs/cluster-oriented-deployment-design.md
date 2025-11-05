# Cluster-Oriented Deployment Design

## Overview

This design enables selective deployment of codebases to clusters by creating deployment marker files. When you create a file in an environment folder, that codebase will be deployed to that environment.

**Key principle**: File existence = deployment enabled

## Directory Structure

```
deployments/
├── eu-central-1/
│   ├── euc1-dev/
│   │   ├── frontend-app.yaml
│   │   └── payment-service.yaml
│   ├── euc1-staging/
│   │   └── frontend-app.yaml
│   └── euc1-prod/
│       └── frontend-app.yaml
└── eu-south-1/
    ├── eus1-dev/
    │   └── payment-service.yaml
    └── eus1-prod/
        └── payment-service.yaml
```

### Path Structure

`deployments/{region}/{cluster}/{codebase}.yaml`

- **{region}**: AWS region (eu-central-1, eu-south-1)
- **{cluster}**: Cluster name (euc1-dev, euc1-staging, euc1-prod, etc.)
- **{codebase}**: Codebase/application name (frontend-app, payment-service, etc.)

### Path Parsing in ApplicationSet

The ApplicationSet uses Go template path indexing:
- `{{.path[0]}}` = `deployments`
- `{{.path[1]}}` = region (e.g., `eu-central-1`)
- `{{.path[2]}}` = cluster name (e.g., `euc1-dev`)
- `{{.path.basenameNormalized}}` = codebase name (e.g., `frontend-app`)

## Deployment Marker File Format

### Standard File

```yaml
# deployments/eu-central-1/euc1-dev/frontend-app.yaml
cluster:
  # Required: Kubernetes API server URL
  server: https://kubernetes.default.svc

  # Required: Target namespace for ArgoCD application
  namespace: argocd

  # Required: Environment name (used for targetRevision lookup in codebase.yaml)
  environment: development

deployment:
  # Optional: Override targetRevision from codebase.yaml
  # Use this for hotfixes, feature branches, or environment-specific versions
  # targetRevision: feature-branch-xyz

  # Optional: Additional values file (future extension)
  # extraValuesFile: special-config.yaml
```

### Minimal File (Most Common)

```yaml
# deployments/eu-central-1/euc1-staging/frontend-app.yaml
cluster:
  server: https://euc1-staging-k8s.example.com
  namespace: argocd
  environment: staging

deployment: {}
```

### Emergency Hotfix Override

```yaml
# deployments/eu-central-1/euc1-prod/frontend-app.yaml
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  # Override to deploy hotfix instead of regular production version
  targetRevision: hotfix/security-patch-v1.2.1
```

## ApplicationSet Configuration

The ApplicationSet uses a **two-generator Matrix**:

1. **Generator 1**: Git File Generator scanning `deployments/*/*/*.yaml`
   - Provides: region (from path), cluster name (from path), codebase name (from filename), cluster config (from file content)

2. **Generator 2**: Git File Generator loading codebase configuration
   - Path: `codebases/{{.path.basenameNormalized}}/codebase.yaml`
   - Provides: codebase metadata (repoURL, chartPath, targetRevisions matrix)

### How It Works

```yaml
generators:
  - matrix:
      generators:
        # Scan deployment markers
        - git:
            files:
              - path: 'deployments/*/*/*.yaml'

        # Load codebase config using codebase name from deployment file
        - git:
            files:
              - path: 'codebases/{{.path.basenameNormalized}}/codebase.yaml'
```

The Matrix Generator creates Applications only for combinations where both:
1. A deployment marker file exists (`deployments/{region}/{cluster}/{codebase}.yaml`)
2. A codebase configuration exists (`codebases/{codebase}/codebase.yaml`)

## Workflow Examples

### Example 1: New Application - Progressive Rollout

#### Step 1: Define the Codebase (Once)

```bash
# Create codebase configuration
cat > codebases/new-api/codebase.yaml <<EOF
codebase:
  name: new-api
  repoURL: https://github.com/myorg/new-api
  chartPath: deploy-templates
  namespace: backend

  # Define versions for all regions and environments
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
mkdir -p codebases/new-api/values/eu-central-1
cat > codebases/new-api/values/eu-central-1/development.yaml <<EOF
replicaCount: 1
resources:
  requests:
    cpu: 100m
    memory: 128Mi
env:
  - name: ENVIRONMENT
    value: development
  - name: LOG_LEVEL
    value: debug
EOF

mkdir -p codebases/new-api/values/eu-central-1
cat > codebases/new-api/values/eu-central-1/staging.yaml <<EOF
replicaCount: 2
resources:
  requests:
    cpu: 200m
    memory: 256Mi
env:
  - name: ENVIRONMENT
    value: staging
  - name: LOG_LEVEL
    value: info
EOF
```

**Result**: No Applications created yet. Codebase is defined but not deployed anywhere.

#### Step 2: Deploy to Development

```bash
# Create deployment marker for dev environment
mkdir -p deployments/eu-central-1/euc1-dev

cat > deployments/eu-central-1/euc1-dev/new-api.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
EOF

git add deployments/eu-central-1/euc1-dev/new-api.yaml
git commit -m "Deploy new-api to euc1-dev"
git push
```

**Result**:
- ArgoCD detects new deployment file
- Application `euc1-dev-new-api` is created
- Deploys from `main` branch (as defined in codebase.yaml)
- Uses values from `codebases/new-api/values/eu-central-1/development.yaml`

#### Step 3: Test, Then Promote to Staging

```bash
# After testing in dev, enable staging deployment
mkdir -p deployments/eu-central-1/euc1-staging

cat > deployments/eu-central-1/euc1-staging/new-api.yaml <<EOF
cluster:
  server: https://euc1-staging-k8s.example.com
  namespace: argocd
  environment: staging

deployment: {}
EOF

git add deployments/eu-central-1/euc1-staging/new-api.yaml
git commit -m "Promote new-api to euc1-staging"
git push
```

**Result**:
- Application `euc1-staging-new-api` is created
- Deploys from `v1.0.0` tag (as defined in codebase.yaml)
- Uses values from `codebases/new-api/values/eu-central-1/staging.yaml`

#### Step 4: Eventually Promote to Production

```bash
# After successful staging testing
mkdir -p deployments/eu-central-1/euc1-prod

cat > deployments/eu-central-1/euc1-prod/new-api.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment: {}
EOF

git add deployments/eu-central-1/euc1-prod/new-api.yaml
git commit -m "Promote new-api to production"
git push
```

### Example 2: Multi-Region Deployment

```bash
# Deploy to eu-south-1 development
mkdir -p deployments/eu-south-1/eus1-dev

cat > deployments/eu-south-1/eus1-dev/new-api.yaml <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: development

deployment: {}
EOF

git add deployments/eu-south-1/eus1-dev/new-api.yaml
git commit -m "Deploy new-api to eus1-dev"
git push
```

**Result**: Independent deployment in eu-south-1 region, managed by that region's ArgoCD instance.

### Example 3: Emergency Hotfix

```bash
# Override production version for critical security fix
cat > deployments/eu-central-1/euc1-prod/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  # Deploy hotfix branch immediately
  targetRevision: hotfix/security-cve-2024-001
EOF

git add deployments/eu-central-1/euc1-prod/frontend-app.yaml
git commit -m "HOTFIX: Deploy security patch to production"
git push
```

**Result**:
- Only `euc1-prod-frontend-app` uses the hotfix branch
- All other clusters (staging, dev, other regions) continue using their configured versions
- When hotfix is merged and tagged, remove the override to return to normal versioning

### Example 4: Add New Cluster

```bash
# Step 1: Update targetRevisions in codebases that should deploy to QA
# Edit codebases/*/codebase.yaml to add qa: version under eu-central-1
# Example: add "qa: v1.0.0" under targetRevisions.eu-central-1

# Step 2: Enable deployments for selected codebases
mkdir -p deployments/eu-central-1/euc1-qa

cat > deployments/eu-central-1/euc1-qa/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-qa-k8s.example.com
  namespace: argocd
  environment: qa

deployment: {}
EOF

cat > deployments/eu-central-1/euc1-qa/payment-service.yaml <<EOF
cluster:
  server: https://euc1-qa-k8s.example.com
  namespace: argocd
  environment: qa

deployment: {}
EOF

git add .
git commit -m "Add QA cluster and deploy selected applications"
git push
```

**Result**: Only frontend-app and payment-service deploy to QA. No forced deployments.

### Example 5: Rollback

```bash
# Rollback by changing targetRevision in deployment file
cat > deployments/eu-central-1/euc1-prod/frontend-app.yaml <<EOF
cluster:
  server: https://euc1-prod-k8s.example.com
  namespace: argocd
  environment: production

deployment:
  # Rollback to previous stable version
  targetRevision: v1.1.9
EOF

git add deployments/eu-central-1/euc1-prod/frontend-app.yaml
git commit -m "Rollback frontend-app in production to v1.1.9"
git push
```

### Example 6: Disable Deployment

```bash
# Remove deployment marker file to disable deployment
git rm deployments/eu-central-1/euc1-staging/old-service.yaml
git commit -m "Disable old-service in staging"
git push
```

**Result**: ArgoCD deletes the Application, and the service is undeployed from staging.

## Key Benefits

✅ **Independent Addition**: Add clusters and codebases independently without forcing deployments

✅ **Progressive Deployment**: Control exactly which codebase deploys to which cluster by creating files

✅ **File-Based Trigger**: Creating a file in the environment folder enables deployment - intuitive and Git-tracked

✅ **No Cartesian Product**: Only combinations with deployment files generate Applications

✅ **Per-Cluster Version Control**: targetRevisions matrix in codebase.yaml + deployment overrides

✅ **Emergency Overrides**: Deployment file can override version for specific cluster

✅ **Native ArgoCD**: Uses only Matrix Generator and Git File Generator - no custom plugins

✅ **Clear Audit Trail**: Git history shows exactly when each deployment was enabled/disabled

✅ **Multi-Region Support**: Each region has independent deployment decisions

## Version Resolution Logic

The ApplicationSet resolves `targetRevision` with the following precedence:

1. **Deployment Override** (highest priority): `deployment.targetRevision` in deployment marker file
2. **Codebase Matrix** (default): `codebase.targetRevisions[region][environment]` from codebase.yaml

Template logic:
```yaml
targetRevision: '{{if .deployment.targetRevision}}{{.deployment.targetRevision}}{{else}}{{index (index .codebase.targetRevisions .path[1]) .cluster.environment}}{{end}}'
```

## Comparison with Other Approaches

### vs. List Generator
- **List Generator**: Can't iterate over arrays in YAML files, requires one file per combination
- **This Design**: Matrix naturally combines files, clean separation of concerns

### vs. Directory-Based Tiers
- **Directory Tiers**: `codebases-dev/`, `codebases-staging/`, `codebases-prod/` - organized by maturity
- **This Design**: Organized by cluster, more flexible for per-cluster decisions

### vs. Pure Matrix (Previous Design)
- **Pure Matrix**: Creates Applications for ALL cluster × codebase combinations
- **This Design**: Only creates Applications where deployment file exists

## Migration Path

### From Current Structure

1. Keep existing `codebases/` directory structure unchanged
2. Create `deployments/` directory with marker files for currently deployed applications
3. Update ApplicationSet to use new two-generator Matrix
4. Test in development cluster first
5. Roll out region by region

### Automated Migration Script

```bash
#!/bin/bash
# migrate-to-cluster-oriented.sh

# For each codebase, create deployment marker files for all clusters
for codebase in codebases/*/; do
  codebase_name=$(basename "$codebase")

  # Read targetRevisions to know which clusters to deploy to
  # This is a simplified example - actual implementation would parse YAML

  for region in eu-central-1 eu-south-1; do
    for env in development staging production; do
      cluster_name="${region:0:3}${region:12:1}-${env:0:4}"

      mkdir -p "deployments/$region/$cluster_name"
      cat > "deployments/$region/$cluster_name/$codebase_name.yaml" <<EOF
cluster:
  server: https://kubernetes.default.svc
  namespace: argocd
  environment: $env

deployment: {}
EOF
    done
  done
done
```

## Future Extensions

### 1. Progressive Sync (RollingSync)

Enable gradual rollout across clusters:

```yaml
# In ApplicationSet spec
strategy:
  type: RollingSync
  rollingSync:
    steps:
      - matchExpressions:
          - key: environment
            operator: In
            values: [development]
      - matchExpressions:
          - key: environment
            operator: In
            values: [staging]
      - matchExpressions:
          - key: environment
            operator: In
            values: [production]
```

### 2. Extra Values Files

```yaml
# In deployment marker file
deployment:
  extraValuesFile: special-config.yaml
```

Reference in ApplicationSet:
```yaml
valueFiles:
  - values.yaml
  - $values/codebases/{{.path.basenameNormalized}}/values/{{.path[1]}}/{{.cluster.environment}}.yaml
  {{- if .deployment.extraValuesFile}}
  - $values/codebases/{{.path.basenameNormalized}}/values/{{.deployment.extraValuesFile}}
  {{- end}}
```

### 3. Deployment Windows

```yaml
# In deployment marker file
deployment:
  targetRevision: v2.0.0
  syncWindow:
    schedule: "0 2 * * *"  # Deploy at 2 AM daily
    duration: 2h
```

### 4. Canary Deployments

```yaml
# In deployment marker file
deployment:
  targetRevision: v2.0.0
  canary:
    enabled: true
    steps:
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

## Troubleshooting

### Application Not Created

**Problem**: Created deployment file but Application doesn't appear

**Checks**:
1. Verify file path follows pattern: `deployments/{region}/{cluster}/{codebase}.yaml`
2. Confirm codebase.yaml exists at `codebases/{codebase}/codebase.yaml`
3. Check ArgoCD ApplicationSet controller logs
4. Verify Git repository is synced in ArgoCD

### Wrong Version Deployed

**Problem**: Application deploys different version than expected

**Checks**:
1. Check if deployment file has `targetRevision` override
2. Verify `environment` in deployment file matches expected environment
3. Confirm targetRevisions matrix in codebase.yaml has entry for this region+environment
4. Check Application manifest: `kubectl get app {cluster}-{codebase} -n argocd -o yaml`

### Sync Failures

**Problem**: Application created but sync fails

**Checks**:
1. Verify `cluster.server` URL is correct and reachable
2. Check namespace exists or `CreateNamespace=true` is set
3. Verify values file exists: `codebases/{codebase}/values/{region}/{environment}.yaml`
4. Review Application events: `kubectl describe app {cluster}-{codebase} -n argocd`

## Best Practices

1. **Naming Conventions**: Use consistent cluster naming (e.g., `euc1-dev`, `eus1-prod`)

2. **Minimal Deployment Files**: Only include overrides when necessary, keep most files minimal

3. **Git Commit Messages**: Be descriptive about deployment intentions
   - Good: "Deploy payment-service v2.1.0 to euc1-staging for load testing"
   - Bad: "Update deployment file"

4. **Progressive Rollout**: Always follow dev → staging → production progression

5. **Version Pinning**: Use tags/versions in production, `main` branch acceptable for dev

6. **Documentation**: Update codebase README when adding new deployment dependencies

7. **Cleanup**: Remove deployment files when applications are decommissioned

8. **Review Process**: Require PR reviews for production deployment file changes

9. **Testing**: Test deployment changes in dev before promoting to higher environments

10. **Monitoring**: Set up alerts for Application OutOfSync or Degraded states
