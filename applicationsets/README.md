# ApplicationSets

This directory contains ApplicationSet templates used to generate ArgoCD Applications.

## Overview

The ApplicationSets in this directory are not applied directly. Instead, they are referenced by the root ApplicationSet and instantiated once per cluster.

## Files

### cluster-apps.yaml

**Purpose**: Template for generating Applications from codebases

**How it's used**:
1. Root ApplicationSet (`bootstrap/root-appset.yaml`) scans the `clusters/` directory
2. For each cluster found, it creates an ApplicationSet from this template
3. The cluster name is passed as a parameter via Helm

**Generated ApplicationSets**:
- `dev-apps` (for dev cluster)
- `staging-apps` (for staging cluster)
- `prod-apps` (for prod cluster)
- One per cluster in `clusters/` directory

**What it does**:
- Uses a Matrix generator to combine:
  - Cluster configuration from `clusters/<cluster>/config.yaml`
  - All codebase configurations from `codebases/*/codebase.yaml`
- Generates one Application per codebase for the specific cluster
- Configures Applications to use multiple sources:
  - Remote Helm chart from application repository
  - Local values overrides from this repository

## How Applications Are Generated

### Input Parameters

The cluster ApplicationSet receives parameters from the root ApplicationSet:

```yaml
parameters:
  - name: clusterName
    value: dev  # or staging, prod, etc.
  - name: repoURL
    value: https://github.com/YourOrg/argocd-app-of-app.git
  - name: revision
    value: main
```

### Generator Logic

The ApplicationSet uses a matrix generator:

```yaml
generators:
  - matrix:
      generators:
        # Generator 1: Read cluster config
        - git:
            files:
              - path: 'clusters/{{.Values.clusterName}}/config.yaml'

        # Generator 2: Read all codebase configs
        - git:
            files:
              - path: 'codebases/*/codebase.yaml'
```

This creates a cross-product:
- For cluster `dev`
- And codebases: `frontend-app`, `backend-api`, `data-processor`
- Generates: `dev-frontend-app`, `dev-backend-api`, `dev-data-processor`

### Application Template

Each generated Application has this structure:

```yaml
metadata:
  name: '{{.codebase.name}}-{{.cluster.name}}'

spec:
  sources:
    # Remote Helm chart
    - repoURL: '{{.codebase.repoURL}}'
      path: '{{.codebase.chartPath}}'
      helm:
        valueFiles:
          - values.yaml
          - $values/codebases/{{.codebase.name}}/values/{{.cluster.name}}.yaml

    # Local values repository
    - repoURL: '{{.Values.repoURL}}'
      ref: values

  destination:
    server: '{{.cluster.server}}'
    namespace: '{{.codebase.namespace}}'
```

## Customization

### Modifying Application Defaults

To change settings for all generated Applications, edit `cluster-apps.yaml`:

**Example: Change sync policy**

```yaml
template:
  spec:
    syncPolicy:
      automated:
        prune: false  # Changed from true
        selfHeal: true
```

**Example: Add resource tracking**

```yaml
template:
  spec:
    syncPolicy:
      syncOptions:
        - CreateNamespace=true
        - PrunePropagationPolicy=foreground  # Added
        - PruneLast=true                     # Added
```

### Adding Filters

To deploy only specific codebases to specific clusters, add selectors:

**Example: Only deploy apps with matching environment label**

```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: 'clusters/{{.Values.clusterName}}/config.yaml'
        - git:
            files:
              - path: 'codebases/*/codebase.yaml'
      # Add selector
      selector:
        matchExpressions:
          - key: codebase.team
            operator: In
            values: [platform-team]
```

### Using Different Generators

**List Generator** (for explicit combinations):

```yaml
generators:
  - list:
      elements:
        - codebase: frontend-app
          cluster: dev
        - codebase: frontend-app
          cluster: staging
        # Explicit list of combinations
```

**Cluster Generator** (use ArgoCD's cluster registry):

```yaml
generators:
  - matrix:
      generators:
        - cluster:
            selector:
              matchLabels:
                environment: production
        - git:
            files:
              - path: 'codebases/*/codebase.yaml'
```

## Troubleshooting

### ApplicationSet Not Creating Applications

**Check ApplicationSet status**:
```bash
kubectl describe applicationset dev-apps -n argocd
```

**Look for**:
- Generator errors
- Template rendering errors
- Git repository access issues

**Force refresh**:
```bash
kubectl patch applicationset dev-apps -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'
```

### Applications Not Using Latest Values

**Possible causes**:
1. Values file not committed to Git
2. ArgoCD hasn't refreshed from Git
3. Application sync not triggered

**Solution**:
```bash
# Check Git has latest
git log -- codebases/frontend-app/values/dev.yaml

# Force ApplicationSet refresh
kubectl patch applicationset dev-apps -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"now"}}}'

# Force Application sync
argocd app sync dev-frontend-app
```

### Template Rendering Errors

**Check logs**:
```bash
kubectl logs -n argocd deployment/argocd-applicationset-controller
```

**Common issues**:
- Missing required fields in codebase.yaml
- Invalid Go template syntax
- Missing values in cluster config

## Advanced Patterns

### Multi-Environment Progressive Rollout

Add a generator to only deploy tested versions to production:

```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: 'clusters/{{.Values.clusterName}}/config.yaml'
        - git:
            files:
              - path: 'codebases/*/codebase.yaml'

template:
  spec:
    source:
      targetRevision: |
        {{- if eq .cluster.environment "production" }}
        {{ .codebase.prodVersion | default .codebase.targetRevision }}
        {{- else }}
        {{ .codebase.targetRevision }}
        {{- end }}
```

### Conditional Features

Enable features based on cluster:

```yaml
template:
  spec:
    source:
      helm:
        parameters:
          - name: monitoring.enabled
            value: {{ eq .cluster.environment "production" | quote }}
```

### Per-Team ApplicationSets

Instead of per-cluster, create per-team ApplicationSets:

```yaml
# In bootstrap, scan teams/ instead of clusters/
generators:
  - git:
      directories:
        - path: teams/*
```

## Validation

### Validate Before Applying

Use ArgoCD CLI to validate ApplicationSet:

```bash
argocd appset get dev-apps
```

### Dry Run

Preview what Applications would be created:

```bash
kubectl get applicationset dev-apps -n argocd -o yaml | \
  yq eval '.status.resources'
```

### Lint YAML

```bash
yamllint applicationsets/cluster-apps.yaml
```

## References

- [ApplicationSet Generators](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators/)
- [Go Template Functions](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/GoTemplate/)
- [Matrix Generator](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Matrix/)
- [Git File Generator](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Git/)
