# Onboarding Guide

This guide walks you through onboarding new applications and clusters to the ArgoCD apps-of-apps system.

## Prerequisites

- Git repository access (write permissions)
- Understanding of your application's Helm chart structure
- Cluster connection details (for new clusters)

## Onboarding a New Application (Codebase)

### Step 1: Prepare Your Application Repository

Your application repository should have:
- A Helm chart in the `deploy-templates` directory (or another path you specify)
- Default `values.yaml` with all configurable parameters

Example repository structure:
```
my-app/
├── src/              # Application source code
├── deploy-templates/ # Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── Dockerfile
```

### Step 2: Create Codebase Directory

Create a directory for your application:

```bash
cd argocd-app-of-app
mkdir -p codebases/my-app/values
```

### Step 3: Create codebase.yaml

Create `codebases/my-app/codebase.yaml`:

```yaml
codebase:
  # Application name (must be unique)
  name: my-app

  # Repository URL where Helm chart is located
  repoURL: https://github.com/myorg/my-app

  # Path to Helm chart within the repository
  chartPath: deploy-templates

  # Target revision (branch, tag, or commit SHA)
  targetRevision: main

  # Kubernetes namespace for deployment
  # If not specified, defaults to the codebase name
  namespace: my-app

  # Optional metadata
  description: "My awesome application"
  team: "my-team"
```

### Step 4: Create Values Overrides

Create environment-specific values for each cluster:

```bash
touch codebases/my-app/values/dev.yaml
touch codebases/my-app/values/staging.yaml
touch codebases/my-app/values/prod.yaml
```

Example `codebases/my-app/values/dev.yaml`:

```yaml
# Development environment overrides
image:
  tag: "dev-latest"
  pullPolicy: Always

replicaCount: 1

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: true
  host: my-app-dev.example.com

env:
  - name: ENVIRONMENT
    value: "development"
  - name: LOG_LEVEL
    value: "debug"
```

### Step 5: Commit and Push

```bash
git add codebases/my-app/
git commit -m "Onboard my-app application"
git push origin main
```

### Step 6: Verify Deployment

ArgoCD will automatically detect the new codebase and create Applications for all clusters:

```bash
# Check ApplicationSets
kubectl get applicationsets -n argocd

# Check generated Applications
kubectl get applications -n argocd | grep my-app
```

You should see:
- `my-app-dev`
- `my-app-staging`
- `my-app-prod`

### Step 7: Monitor Sync Status

```bash
# Watch application sync status
kubectl get application my-app-dev -n argocd -w

# Or use ArgoCD CLI
argocd app get my-app-dev
argocd app sync my-app-dev
```

## Onboarding a New Cluster

### Step 1: Register Cluster with ArgoCD

First, register your cluster with ArgoCD:

```bash
# Login to ArgoCD
argocd login <argocd-server>

# Add cluster
argocd cluster add <context-name>

# Verify
argocd cluster list
```

### Step 2: Create Cluster Directory

```bash
mkdir -p clusters/qa
```

### Step 3: Create config.yaml

Create `clusters/qa/config.yaml`:

```yaml
cluster:
  # Cluster name (must match ArgoCD cluster name)
  name: qa

  # Kubernetes API server URL
  # Get this from: kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
  server: https://qa-k8s.example.com

  # Namespace where ArgoCD manages resources
  namespace: argocd

  # Environment label
  environment: qa
```

### Step 4: Create Values for All Codebases

For each existing codebase, create a values file for the new cluster:

```bash
# List all codebases
ls codebases/

# For each codebase, create a values file
touch codebases/frontend-app/values/qa.yaml
touch codebases/backend-api/values/qa.yaml
touch codebases/data-processor/values/qa.yaml
```

Populate each file with appropriate overrides for the QA environment.

### Step 5: Commit and Push

```bash
git add clusters/qa codebases/*/values/qa.yaml
git commit -m "Add QA cluster"
git push origin main
```

### Step 6: Verify

ArgoCD will create a new ApplicationSet for the QA cluster:

```bash
# Check for new ApplicationSet
kubectl get applicationset qa-apps -n argocd

# Check for generated Applications
kubectl get applications -n argocd | grep qa
```

## Common Patterns

### Different Chart Paths

If your Helm chart is not in `deploy-templates`:

```yaml
codebase:
  name: my-app
  repoURL: https://github.com/myorg/my-app
  chartPath: kubernetes/helm/my-app  # Custom path
  targetRevision: main
```

### Using Git Tags for Production

```yaml
codebase:
  name: my-app
  repoURL: https://github.com/myorg/my-app
  chartPath: deploy-templates
  targetRevision: v1.2.3  # Git tag instead of branch
```

### Multiple Namespaces per Codebase

If you want to deploy the same app to different namespaces per cluster, use values overrides:

In `codebase.yaml`, omit the namespace:
```yaml
codebase:
  name: my-app
  # namespace not specified
```

In `values/dev.yaml`, add:
```yaml
namespace: my-app-dev
```

In `values/prod.yaml`, add:
```yaml
namespace: my-app-prod
```

### Conditional Deployments

To deploy a codebase only to specific clusters, you can:

1. Only create values files for desired clusters
2. Modify ApplicationSet generators to filter based on labels

Example: Deploy only to dev and staging:
```bash
# Only create these files
touch codebases/my-app/values/dev.yaml
touch codebases/my-app/values/staging.yaml
# Don't create prod.yaml
```

### Using External Helm Charts

If your chart is in a Helm repository instead of Git:

You'll need to modify the ApplicationSet template in `applicationsets/cluster-apps.yaml` to support Helm repositories. Contact your platform team for guidance.

## Troubleshooting

### Application Not Created

**Symptom**: No Application appears after onboarding

**Possible Causes**:
1. ApplicationSet hasn't synced yet
2. Values file missing for cluster
3. Syntax error in YAML files

**Solution**:
```bash
# Force ApplicationSet refresh
kubectl patch applicationset <cluster>-apps -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'

# Check ApplicationSet status
kubectl describe applicationset <cluster>-apps -n argocd

# Verify files exist
ls -la codebases/my-app/
ls -la codebases/my-app/values/
```

### Application Sync Fails

**Symptom**: Application created but sync fails

**Possible Causes**:
1. Invalid Helm chart
2. Missing dependencies
3. Resource quota exceeded
4. RBAC issues

**Solution**:
```bash
# Check Application details
argocd app get <app-name>

# View sync errors
kubectl describe application <app-name> -n argocd

# Check application logs
argocd app logs <app-name>
```

### Values Override Not Applied

**Symptom**: Values from override file not being used

**Possible Causes**:
1. File not committed to Git
2. Wrong file path or name
3. YAML syntax error

**Solution**:
```bash
# Verify file is in Git
git log -- codebases/my-app/values/dev.yaml

# Validate YAML syntax
yamllint codebases/my-app/values/dev.yaml

# Check Application manifest
kubectl get application my-app-dev -n argocd -o yaml
```

### Multiple Sources Error

**Symptom**: Error about multiple sources not supported

**Solution**: Ensure you're using ArgoCD v2.6+ which supports multiple sources. Check version:
```bash
argocd version
```

## Best Practices

1. **Version Control**: Always commit changes to Git before expecting updates
2. **Small Batches**: Onboard one application at a time to identify issues quickly
3. **Test in Dev**: Verify in dev cluster before rolling to staging/prod
4. **Use Branches**: Create feature branches for onboarding, merge after verification
5. **Documentation**: Update codebase.yaml with description and team info
6. **Validation**: Use `yamllint` and `helm lint` to validate files before committing
7. **Gradual Rollout**: Deploy to dev → staging → prod with verification at each step

## Checklist

### New Application Onboarding

- [ ] Application has Helm chart in repository
- [ ] Create `codebases/<app-name>/` directory
- [ ] Create `codebase.yaml` with correct repository URL
- [ ] Create values override for each cluster
- [ ] Validate YAML syntax
- [ ] Commit and push to Git
- [ ] Verify Applications created
- [ ] Verify Applications sync successfully
- [ ] Test application functionality in dev
- [ ] Document in team wiki/runbook

### New Cluster Onboarding

- [ ] Cluster registered with ArgoCD
- [ ] Create `clusters/<cluster-name>/` directory
- [ ] Create `config.yaml` with correct server URL
- [ ] Create values overrides for all codebases
- [ ] Validate YAML syntax
- [ ] Commit and push to Git
- [ ] Verify ApplicationSet created
- [ ] Verify Applications created for all codebases
- [ ] Verify sync status
- [ ] Update cluster documentation

## Getting Help

If you encounter issues:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review ArgoCD logs: `kubectl logs -n argocd deployment/argocd-applicationset-controller`
3. Consult the [Architecture documentation](./architecture.md)
4. Contact the platform team
