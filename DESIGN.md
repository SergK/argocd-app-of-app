# ArgoCD Apps-of-Apps Design

## Overview

This repository implements an apps-of-apps pattern using ArgoCD ApplicationSet to manage deployments across multiple codebases and clusters. The design follows Akuity's GitOps best practices and enables:

- Automated Application generation based on codebase and cluster combinations
- Remote Helm chart references from codebase repositories
- Local values overrides per codebase and cluster
- Self-service application onboarding
- Centralized cluster and codebase management

## Repository Structure

```bash
argocd-app-of-app/
├── README.md                           # Repository overview and getting started
├── DESIGN.md                           # This design document
│
├── bootstrap/                          # Bootstrap ArgoCD with the root ApplicationSet
│   └── root-appset.yaml               # Root ApplicationSet that creates cluster ApplicationSets
│
├── clusters/                           # Cluster-specific configurations
│   ├── dev/
│   │   └── config.yaml                # Cluster metadata (name, server URL, etc.)
│   ├── staging/
│   │   └── config.yaml
│   └── prod/
│       └── config.yaml
│
├── codebases/                          # Codebase definitions and configurations
│   ├── frontend-app/
│   │   ├── codebase.yaml              # Codebase metadata (repo URL, chart path)
│   │   └── values/                    # Values overrides per cluster
│   │       ├── dev.yaml
│   │       ├── staging.yaml
│   │       └── prod.yaml
│   ├── backend-api/
│   │   ├── codebase.yaml
│   │   └── values/
│   │       ├── dev.yaml
│   │       ├── staging.yaml
│   │       └── prod.yaml
│   └── data-processor/
│       ├── codebase.yaml
│       └── values/
│           ├── dev.yaml
│           ├── staging.yaml
│           └── prod.yaml
│
├── applicationsets/                    # ApplicationSet definitions
│   ├── cluster-apps.yaml              # Generates ApplicationSets per cluster
│   └── README.md                      # Documentation for ApplicationSets
│
└── docs/                               # Additional documentation
    ├── onboarding.md                  # How to onboard new codebases/clusters
    └── architecture.md                # Architecture decisions and patterns
```

## Design Components

### 1. Bootstrap Layer

The `bootstrap/root-appset.yaml` is the entry point. This ApplicationSet:

- Uses Git Directory generator to scan the `clusters/` directory
- Creates one ApplicationSet per cluster
- Each cluster ApplicationSet will scan codebases and create Applications

### 2. Cluster Definitions

Each cluster has a configuration file at `clusters/<cluster-name>/config.yaml`:

```yaml
# clusters/dev/config.yaml
cluster:
  name: dev
  server: https://kubernetes.default.svc
  namespace: argocd
```

### 3. Codebase Definitions

Each codebase has:

- A `codebase.yaml` file with metadata
- A `values/` directory with cluster-specific overrides

Example `codebases/frontend-app/codebase.yaml`:

```yaml
codebase:
  name: frontend-app
  repoURL: https://github.com/myorg/frontend-app
  chartPath: deploy-templates
  targetRevision: main
  namespace: frontend
```

### 4. ApplicationSet Pattern

The design uses a layered ApplicationSet approach:

1. **Root ApplicationSet** (`bootstrap/root-appset.yaml`):
   - Scans `clusters/` directory
   - Creates per-cluster ApplicationSets

2. **Cluster ApplicationSet** (`applicationsets/cluster-apps.yaml`):
   - Scans `codebases/` directory
   - For each codebase, creates an Application targeting that cluster
   - Merges remote Helm chart with local values overrides

### 5. Values Override Strategy

Values are organized hierarchically:

- Remote Helm chart contains default values
- `codebases/<codebase>/values/<cluster>.yaml` provides cluster-specific overrides
- ArgoCD merges these using Helm's values precedence

## Application Generation Flow

```text
┌─────────────────────────────────────────────────────────────┐
│  Bootstrap Root ApplicationSet                              │
│  (bootstrap/root-appset.yaml)                              │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  │ Scans clusters/ directory
                  ▼
┌─────────────────────────────────────────────────────────────┐
│  Creates per-cluster ApplicationSets                        │
│  - dev-apps ApplicationSet                                  │
│  - staging-apps ApplicationSet                              │
│  - prod-apps ApplicationSet                                 │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  │ Each scans codebases/ directory
                  ▼
┌─────────────────────────────────────────────────────────────┐
│  Generates Applications (codebase × cluster matrix)         │
│  - dev-frontend-app                                         │
│  - staging-frontend-app                                     │
│  - prod-frontend-app                                        │
│  - dev-backend-api                                          │
│  - staging-backend-api                                      │
│  - prod-backend-api                                         │
│  - dev-data-processor                                       │
│  - staging-data-processor                                   │
│  - prod-data-processor                                      │
└─────────────────────────────────────────────────────────────┘
```

## Key Features

### Self-Service Onboarding

To add a new codebase:

1. Create directory under `codebases/<new-codebase>/`
2. Add `codebase.yaml` with repository and chart information
3. Add values overrides in `codebases/<new-codebase>/values/<cluster>.yaml`
4. Commit and push - ApplicationSet automatically creates Applications

To add a new cluster:

1. Create directory under `clusters/<new-cluster>/`
2. Add `config.yaml` with cluster information
3. Commit and push - ApplicationSet automatically creates cluster ApplicationSet

### Values Override Mechanism

Applications use multiple values files:

1. Default values from remote Helm chart
2. Cluster-specific overrides from this repository

Example Application spec:

```yaml
spec:
  sources:
    - repoURL: https://github.com/myorg/frontend-app
      targetRevision: main
      path: deploy-templates
      helm:
        # Use short codebase name for Helm release
        releaseName: frontend-app
        valueFiles:
          - values.yaml  # From remote chart
          - $values/codebases/frontend-app/values/dev.yaml
    - repoURL: https://github.com/myorg/argocd-app-of-app
      ref: values
```

**Naming Convention**: Applications are named `<cluster>-<codebase>` (e.g., `dev-frontend-app`), but the Helm release name is set to just the codebase name (e.g., `frontend-app`) to keep resource names clean and consistent across clusters.

### Git-Based Automation

All changes are GitOps-driven:

- Add/remove codebases by modifying `codebases/` directory
- Add/remove clusters by modifying `clusters/` directory
- Update values by modifying files in `codebases/*/values/*.yaml`
- No manual Application creation needed

## Best Practices Implemented

Based on Akuity workshops and ArgoCD best practices:

1. **Separation of Concerns**: Cluster configs and codebase configs are separated
2. **DRY Principle**: Remote Helm charts avoid duplication
3. **Git as Source of Truth**: All configuration in Git
4. **Self-Service**: Teams can onboard by adding files
5. **Scalability**: Matrix of codebases × clusters scales automatically
6. **Flexibility**: Per-cluster overrides for environment differences
7. **Centralized Management**: Single repository for all deployments

## Alternative Approaches

### Approach 1: Matrix Generator (Current Design)

- Uses nested generators to create codebase × cluster matrix
- Pros: Automatic combination of all codebases with all clusters
- Cons: Less granular control over which codebase goes to which cluster

### Approach 2: Git File Generator

- Uses a single config file listing all codebase-cluster pairs
- Pros: Explicit control over deployments
- Cons: Manual maintenance of combinations

### Approach 3: List Generator

- Hardcoded list of codebases and clusters
- Pros: Simple and explicit
- Cons: Not GitOps-friendly, requires ApplicationSet changes

**Selected Approach**: Matrix Generator for automation and scalability

## Security Considerations

1. **Repository Access**: This repo needs read access to all codebase repositories
2. **Cluster Credentials**: Managed through ArgoCD cluster secrets
3. **Values Overrides**: Can override sensitive values - use Sealed Secrets or External Secrets Operator
4. **RBAC**: Use ArgoCD Projects to limit what teams can deploy

## Next Steps

1. Implement the directory structure
2. Create bootstrap ApplicationSet
3. Create cluster ApplicationSet template
4. Add sample codebase configurations
5. Document onboarding process
6. Test with sample deployments
