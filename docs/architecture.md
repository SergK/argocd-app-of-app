# Architecture Documentation

## System Overview

This document describes the architecture of the ArgoCD apps-of-apps system, which automates application deployment across multiple Kubernetes clusters.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Git Repository                              │
│                   (argocd-app-of-app)                              │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  clusters/   │  │ codebases/   │  │ bootstrap/   │            │
│  │    dev/      │  │ frontend-app/│  │ root-appset  │            │
│  │    staging/  │  │ backend-api/ │  │              │            │
│  │    prod/     │  │ data-proc/   │  └──────┬───────┘            │
│  └──────────────┘  └──────────────┘         │                     │
└───────────────────────────────────────────────┼──────────────────────┘
                                                │
                                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ArgoCD in Management Cluster                     │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │ Root ApplicationSet (bootstrap/root-appset.yaml)          │    │
│  │ - Scans clusters/ directory                              │    │
│  │ - Generates per-cluster ApplicationSets                  │    │
│  └─────────────┬─────────────────────────────────────────────┘    │
│                │                                                    │
│                ▼                                                    │
│  ┌──────────────────┬──────────────────┬──────────────────┐       │
│  │  dev-apps        │  staging-apps    │  prod-apps       │       │
│  │  ApplicationSet  │  ApplicationSet  │  ApplicationSet  │       │
│  │                  │                  │                  │       │
│  │  Scans:          │  Scans:          │  Scans:          │       │
│  │  codebases/      │  codebases/      │  codebases/      │       │
│  └────────┬─────────┴────────┬─────────┴────────┬─────────┘       │
│           │                  │                  │                  │
│           ▼                  ▼                  ▼                  │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │           Generated Applications                        │      │
│  │                                                          │      │
│  │  dev-frontend-app      staging-frontend-app  ...        │      │
│  │  dev-backend-api       staging-backend-api   ...        │      │
│  │  dev-data-proc         staging-data-proc     ...        │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Application Repositories                         │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │ github.com/myorg/frontend-app                             │    │
│  │   deploy-templates/                                       │    │
│  │     Chart.yaml                                            │    │
│  │     values.yaml          ◄────────┐                       │    │
│  └───────────────────────────────────┼───────────────────────┘    │
│                                       │                            │
│  ┌───────────────────────────────────┼───────────────────────┐    │
│  │ github.com/myorg/backend-api      │                       │    │
│  │   deploy-templates/               │                       │    │
│  │     Chart.yaml                    │                       │    │
│  │     values.yaml          ◄────────┼───┐                   │    │
│  └───────────────────────────────────┼───┼───────────────────┘    │
└─────────────────────────────────────────┼───┼───────────────────────┘
                                          │   │
                    Application pulls    │   │
                    remote Helm chart    │   │
                    and merges with      │   │
                    local values         │   │
                                          │   │
                                          │   │
┌─────────────────────────────────────────┼───┼───────────────────────┐
│                Target Kubernetes Clusters   │                       │
│                                          │   │                       │
│  ┌───────────────────┐  ┌───────────────┼───┼────┐                 │
│  │   Dev Cluster     │  │ Staging       │   │    │                 │
│  │                   │  │               │   │    │                 │
│  │  Namespaces:      │  │  Namespaces:  │   │    │                 │
│  │  - frontend       │  │  - frontend   │   │    │                 │
│  │  - backend        │  │  - backend    │   │    │                 │
│  │  - data-processing│  │  - data-proc..│   │    │                 │
│  └───────────────────┘  └───────────────┴───┴────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. Git Repository (Source of Truth)

The GitOps repository contains all configuration:

#### clusters/
Defines Kubernetes clusters where applications will be deployed.

**Structure**:
```
clusters/
├── dev/
│   └── config.yaml
├── staging/
│   └── config.yaml
└── prod/
    └── config.yaml
```

**Purpose**:
- Store cluster metadata (name, server URL, namespace)
- Enable cluster discovery via Git File generator
- Allow easy addition of new clusters

#### codebases/
Defines applications and their configurations.

**Structure**:
```
codebases/
├── frontend-app/
│   ├── codebase.yaml        # Metadata
│   └── values/
│       ├── dev.yaml         # Dev overrides
│       ├── staging.yaml     # Staging overrides
│       └── prod.yaml        # Prod overrides
└── backend-api/
    └── ...
```

**Purpose**:
- Define application source repositories
- Specify Helm chart location
- Provide cluster-specific value overrides

#### bootstrap/
Entry point for the system.

**Structure**:
```
bootstrap/
└── root-appset.yaml
```

**Purpose**:
- Bootstrap the entire apps-of-apps system
- Create per-cluster ApplicationSets
- Single file to apply to ArgoCD

#### applicationsets/
Templates for ApplicationSet resources.

**Structure**:
```
applicationsets/
└── cluster-apps.yaml
```

**Purpose**:
- Template for generating Applications
- Used by root ApplicationSet to create per-cluster instances
- Contains the matrix logic (codebase × cluster)

### 2. ArgoCD ApplicationSets

#### Root ApplicationSet

**Location**: `bootstrap/root-appset.yaml`

**Generators Used**:
- Git Directory Generator

**What It Does**:
1. Scans `clusters/` directory
2. For each cluster directory found, creates an ApplicationSet
3. Passes cluster name as parameter to child ApplicationSets

**Key Configuration**:
```yaml
generators:
  - git:
      repoURL: https://github.com/YourOrg/argocd-app-of-app.git
      revision: main
      directories:
        - path: clusters/*
```

#### Per-Cluster ApplicationSets

**Location**: Deployed by Root ApplicationSet from `applicationsets/cluster-apps.yaml`

**Generators Used**:
- Matrix Generator combining:
  - Git File Generator (cluster config)
  - Git File Generator (codebase configs)

**What It Does**:
1. Reads cluster configuration from `clusters/<cluster>/config.yaml`
2. Scans all `codebases/*/codebase.yaml` files
3. For each codebase, generates an Application targeting the cluster
4. Configures Application with remote Helm chart + local values

**Key Configuration**:
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
```

### 3. Generated Applications

Each Application represents a deployment of a codebase to a cluster.

**Naming Convention**: `<cluster>-<codebase>` (e.g., `dev-frontend-app`)

**Key Features**:
- Multiple sources (ArgoCD 2.6+)
- Automated sync with self-heal
- Namespace auto-creation

**Application Structure**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-frontend-app
spec:
  sources:
    # Source 1: Remote Helm chart
    - repoURL: https://github.com/myorg/frontend-app
      path: deploy-templates
      helm:
        # Use short name for Helm release (not full Application name)
        releaseName: frontend-app
        valueFiles:
          - values.yaml  # From remote
          - $values/codebases/frontend-app/values/dev.yaml

    # Source 2: Local values overrides
    - repoURL: https://github.com/YourOrg/argocd-app-of-app.git
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
```

**Note**: The Application name is `dev-frontend-app` (cluster-codebase format), but the Helm release name is set to just `frontend-app` to keep resource names clean and consistent across all clusters.

### 4. Application Repositories

Each application has its own Git repository containing:
- Application source code
- Helm chart in `deploy-templates/` (or configured path)
- Default `values.yaml`

**Separation of Concerns**:
- Application repos own the deployment templates
- GitOps repo (this repo) owns environment-specific overrides

## Data Flow

### Initial Bootstrap Flow

1. **Administrator** applies `bootstrap/root-appset.yaml` to ArgoCD:
   ```bash
   kubectl apply -f bootstrap/root-appset.yaml -n argocd
   ```

2. **Root ApplicationSet** scans `clusters/` directory:
   - Finds: `dev/`, `staging/`, `prod/`
   - Creates: `dev-apps`, `staging-apps`, `prod-apps` ApplicationSets

3. **Each Cluster ApplicationSet** scans `codebases/` directory:
   - Finds: `frontend-app/`, `backend-api/`, `data-processor/`
   - Creates matrix: 3 codebases × 3 clusters = 9 Applications

4. **Each Application** syncs to target cluster:
   - Pulls Helm chart from application repository
   - Merges with values from GitOps repository
   - Deploys to cluster

### Adding a New Codebase Flow

1. **Developer** creates `codebases/new-app/` with configuration:
   ```bash
   mkdir -p codebases/new-app/values
   # Create codebase.yaml and values files
   git add codebases/new-app
   git commit -m "Add new-app"
   git push
   ```

2. **ArgoCD** detects Git change:
   - Each cluster ApplicationSet re-evaluates generators
   - New codebase detected in scan

3. **Applications Created**:
   - `dev-new-app`
   - `staging-new-app`
   - `prod-new-app`

4. **Auto-Sync** deploys to clusters

### Adding a New Cluster Flow

1. **Administrator** registers cluster with ArgoCD:
   ```bash
   argocd cluster add qa-cluster
   ```

2. **Administrator** creates `clusters/qa/` with configuration:
   ```bash
   mkdir -p clusters/qa
   # Create config.yaml
   # Create values files for all codebases
   git add clusters/qa codebases/*/values/qa.yaml
   git commit -m "Add QA cluster"
   git push
   ```

3. **Root ApplicationSet** detects new cluster:
   - Creates `qa-apps` ApplicationSet

4. **QA ApplicationSet** scans codebases:
   - Creates Applications for all codebases targeting QA cluster

## Design Decisions

### Why Apps-of-Apps Pattern?

**Advantages**:
- **Automation**: No manual Application creation
- **Scalability**: Easy to add clusters/codebases
- **Consistency**: Same pattern across all deployments
- **Auditability**: All changes tracked in Git

**Alternatives Considered**:
- Manual Application creation: Not scalable
- Single mega-ApplicationSet: Hard to manage, lacks flexibility

### Why Matrix Generator?

**Advantages**:
- Automatically combines codebases with clusters
- Single source of truth for each dimension
- DRY: No duplication of codebase or cluster definitions

**Alternatives Considered**:
- List generator: Requires manual listing of combinations
- Separate ApplicationSets per codebase: Doesn't scale

### Why Multiple Sources?

**Advantages**:
- **Separation of Concerns**: App teams own templates, platform teams own configs
- **Flexibility**: Different revision per environment possible
- **Security**: Centralized sensitive configurations

**Requirements**:
- ArgoCD 2.6+

**Alternatives Considered**:
- Single source with all values: Centralization violates separation of concerns
- Values in application repo: Environment-specific values scattered across repos

### Why Remote Helm Charts?

**Advantages**:
- Application teams control deployment templates
- Version templates with application code
- Update templates without touching GitOps repo

**Alternatives Considered**:
- Helm charts in GitOps repo: Creates coupling and duplication
- Plain manifests: Less flexible than Helm

## Scalability

### Horizontal Scaling

The design scales horizontally:
- **Codebases**: Add unlimited codebases by adding directories
- **Clusters**: Add unlimited clusters by adding directories
- **Combinations**: N codebases × M clusters = N×M Applications

**Example Scaling**:
- 10 codebases × 5 clusters = 50 Applications
- 100 codebases × 10 clusters = 1000 Applications

### Performance Considerations

**Git Repository Size**:
- Codebase configs are small (< 1 KB each)
- Values files are small (typically < 10 KB each)
- Expected: 1000 codebases = ~10 MB repository

**ApplicationSet Processing**:
- Each ApplicationSet runs its generators periodically
- Git File generator is efficient (single clone, file scan)
- Matrix generator multiplies combinations

**ArgoCD Scalability**:
- ArgoCD can handle thousands of Applications
- Consider sharding ArgoCD for > 1000 Applications

## Security

### Access Control

**Repository Access**:
- Read access required for ArgoCD to GitOps repo
- Read access required for ArgoCD to application repos
- Write access for authorized users to commit changes

**RBAC**:
- Use ArgoCD Projects to limit Application permissions
- Restrict which clusters/namespaces teams can deploy to

**Secrets Management**:
- Do NOT commit secrets to Git
- Use External Secrets Operator, Sealed Secrets, or Vault
- Reference secrets from values files

### Git Security

**Branch Protection**:
- Require pull requests for `main` branch
- Require reviews for production changes
- Use signed commits

**Audit Trail**:
- All changes tracked in Git history
- Reviewable in pull requests

## High Availability

### ArgoCD HA

- Run multiple ArgoCD replicas
- Use Redis for HA
- Configure ApplicationSet controller with leader election

### Multi-Cluster Resilience

- Management cluster failure doesn't affect workload clusters
- Applications continue running
- Restore ArgoCD to resume GitOps operations

## Monitoring

### Key Metrics

**ApplicationSet Health**:
```bash
kubectl get applicationset -n argocd
kubectl describe applicationset <name> -n argocd
```

**Application Sync Status**:
```bash
kubectl get applications -n argocd
argocd app list
```

**Git Sync**:
- Monitor ArgoCD logs for Git fetch failures
- Alert on sync failures

### Alerts

Recommended alerts:
- Application sync failed
- Application health degraded
- ApplicationSet error
- Git repository unreachable

## Disaster Recovery

### Backup

**What to Backup**:
- Git repository (primary source of truth)
- ArgoCD configuration
- Cluster registration secrets

**Recovery Steps**:
1. Restore Git repository
2. Reinstall ArgoCD
3. Register clusters
4. Apply `bootstrap/root-appset.yaml`
5. Verify Applications regenerate

### Git as DR

Git history serves as:
- Complete audit trail
- Point-in-time recovery
- Disaster recovery source

## Future Enhancements

### Potential Improvements

1. **Dynamic Namespaces**: Generate namespace per codebase automatically
2. **RBAC Templates**: Auto-generate AppProjects per team
3. **Progressive Delivery**: Integrate with Argo Rollouts
4. **Policy Enforcement**: Add OPA policies for validation
5. **Observability**: Auto-inject monitoring configurations
6. **Multi-Tenancy**: Separate ApplicationSets per team

### Customization Points

The design is extensible:
- Add generators for different discovery methods
- Filter generators with selectors
- Modify templates for different Application specs
- Add webhooks for Git notifications

## References

- [ArgoCD ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/)
- [Akuity GitOps Workshops](https://docs.akuity.io/tutorials/adv-gitops/)
- [Apps-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Multiple Sources Feature](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
