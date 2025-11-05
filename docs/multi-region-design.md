# Multi-Region ArgoCD Architecture Design

## Problem Statement

We need to support two independent ArgoCD instances deployed in two AWS regions:
- **eu-central-1** (Frankfurt)
- **eu-south-1** (Milan)

### Requirements

1. **Independent ArgoCD Instances**: Each region has its own ArgoCD instance
2. **Regional Cluster Management**: Each ArgoCD manages only clusters in its region
3. **Shared Codebases**: Application definitions should not be duplicated
4. **Region-Specific Configurations**: Values can differ per region and cluster
5. **Independent Provisioning**: Each region can be bootstrapped independently

## Proposed Structure

```
argocd-app-of-app/
├── bootstrap/
│   ├── eu-central-1.yaml           # Bootstrap for eu-central-1 ArgoCD
│   ├── eu-south-1.yaml             # Bootstrap for eu-south-1 ArgoCD
│   └── README.md
│
├── regions/
│   ├── eu-central-1/
│   │   └── clusters/
│   │       ├── dev/
│   │       │   └── config.yaml
│   │       ├── staging/
│   │       │   └── config.yaml
│   │       └── prod/
│   │           └── config.yaml
│   └── eu-south-1/
│       └── clusters/
│           ├── dev/
│           │   └── config.yaml
│           ├── staging/
│           │   └── config.yaml
│           └── prod/
│               └── config.yaml
│
├── codebases/
│   ├── frontend-app/
│   │   ├── codebase.yaml
│   │   └── values/
│   │       ├── eu-central-1/
│   │       │   ├── dev.yaml
│   │       │   ├── staging.yaml
│   │       │   └── prod.yaml
│   │       └── eu-south-1/
│   │           ├── dev.yaml
│   │           ├── staging.yaml
│   │           └── prod.yaml
│   ├── backend-api/
│   │   ├── codebase.yaml
│   │   └── values/
│   │       ├── eu-central-1/...
│   │       └── eu-south-1/...
│   └── data-processor/
│       ├── codebase.yaml
│       └── values/
│           ├── eu-central-1/...
│           └── eu-south-1/...
│
├── applicationsets/
│   └── cluster-apps.yaml           # Template used by both regions
│
└── docs/
    ├── architecture.md
    ├── onboarding.md
    └── multi-region.md
```

## Key Design Decisions

### 1. Separate Bootstrap Files per Region

**Why**: Each ArgoCD instance needs to know which region it manages

```yaml
# bootstrap/eu-central-1.yaml
generators:
  - git:
      directories:
        - path: regions/eu-central-1/clusters/*
```

**Deployment**:
```bash
# In eu-central-1 ArgoCD
kubectl apply -f bootstrap/eu-central-1.yaml -n argocd

# In eu-south-1 ArgoCD
kubectl apply -f bootstrap/eu-south-1.yaml -n argocd
```

### 2. Regional Cluster Organization

**Structure**: `regions/{region}/clusters/{cluster-name}/`

**Benefits**:
- Clear separation of regional infrastructure
- Easy to see what clusters belong to which region
- ArgoCD only scans its region's clusters
- Supports different cluster sets per region

### 3. Cluster Naming Convention

**Option A**: Include region in cluster name
```yaml
cluster:
  name: euc1-dev           # Short: eu-central-1 dev
  region: eu-central-1
  server: https://...
```

**Option B**: Keep simple names, rely on path
```yaml
cluster:
  name: dev
  region: eu-central-1
  server: https://...
```

**Recommendation**: Option A (region-prefixed names)
- Ensures unique Application names across all ArgoCD instances
- Makes it clear which region an Application belongs to
- Prevents naming conflicts

**Application naming**: `{cluster.name}-{codebase.name}`
- Example: `euc1-dev-frontend-app`
- Example: `eus1-dev-frontend-app`

### 4. Shared Codebases with Regional Values

**Codebase Definition**: Single `codebase.yaml` (shared across regions)

**Values Organization**: `values/{region}/{cluster}/`

```
codebases/frontend-app/
├── codebase.yaml              # Shared config
└── values/
    ├── eu-central-1/
    │   ├── dev.yaml
    │   ├── staging.yaml
    │   └── prod.yaml
    └── eu-south-1/
        ├── dev.yaml
        ├── staging.yaml
        └── prod.yaml
```

**Benefits**:
- No duplication of codebase definitions
- Region-specific overrides (e.g., different endpoints, resources)
- Cluster-specific overrides within each region
- Clear hierarchy: region → cluster

### 5. ApplicationSet Changes

**Current Template**:
```yaml
valueFiles:
  - values.yaml
  - $values/codebases/{{.codebase.name}}/values/{{.cluster.name}}.yaml
```

**New Template**:
```yaml
valueFiles:
  - values.yaml
  - $values/codebases/{{.codebase.name}}/values/{{.cluster.region}}/{{.cluster.environment}}.yaml
```

Where:
- `{{.cluster.region}}` = `eu-central-1` or `eu-south-1`
- `{{.cluster.environment}}` = `dev`, `staging`, or `prod`

### 6. Matrix Generator Update

**Current**:
```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: 'clusters/{{.Values.clusterName}}/config.yaml'
```

**New**:
```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: 'regions/{{.Values.region}}/clusters/*/config.yaml'
        - git:
            files:
              - path: 'codebases/*/codebase.yaml'
```

## Bootstrap Flow

### For eu-central-1 ArgoCD

1. Apply bootstrap: `kubectl apply -f bootstrap/eu-central-1.yaml`
2. Bootstrap scans: `regions/eu-central-1/clusters/*`
3. Creates ApplicationSets for: `euc1-dev`, `euc1-staging`, `euc1-prod`
4. ApplicationSets scan: `codebases/*/codebase.yaml`
5. Applications use values from: `codebases/*/values/eu-central-1/*.yaml`

### For eu-south-1 ArgoCD

1. Apply bootstrap: `kubectl apply -f bootstrap/eu-south-1.yaml`
2. Bootstrap scans: `regions/eu-south-1/clusters/*`
3. Creates ApplicationSets for: `eus1-dev`, `eus1-staging`, `eus1-prod`
4. ApplicationSets scan: `codebases/*/codebase.yaml`
5. Applications use values from: `codebases/*/values/eu-south-1/*.yaml`

## Application Generation Matrix

### Total Applications

- **Codebases**: 3 (frontend-app, backend-api, data-processor)
- **Regions**: 2 (eu-central-1, eu-south-1)
- **Clusters per region**: 3 (dev, staging, prod)
- **Total**: 3 × 2 × 3 = **18 Applications**

### Application Distribution

**In eu-central-1 ArgoCD**:
- `euc1-dev-frontend-app`
- `euc1-dev-backend-api`
- `euc1-dev-data-processor`
- `euc1-staging-frontend-app`
- `euc1-staging-backend-api`
- `euc1-staging-data-processor`
- `euc1-prod-frontend-app`
- `euc1-prod-backend-api`
- `euc1-prod-data-processor`

**In eu-south-1 ArgoCD**:
- `eus1-dev-frontend-app`
- `eus1-dev-backend-api`
- `eus1-dev-data-processor`
- `eus1-staging-frontend-app`
- `eus1-staging-backend-api`
- `eus1-staging-data-processor`
- `eus1-prod-frontend-app`
- `eus1-prod-backend-api`
- `eus1-prod-data-processor`

## Disaster Recovery & Multi-Region

### Regional Independence

- Each ArgoCD operates independently
- Failure in one region doesn't affect the other
- Each region maintains its own state

### Shared GitOps Repository

- Single source of truth for both regions
- Changes propagate to both regions
- Consistent configuration management

### Region-Specific Configurations

Examples of regional differences:
- AWS endpoints (different per region)
- Resource sizing (cost optimization)
- Data locality requirements
- Compliance configurations

## Onboarding Workflows

### Adding a New Codebase

1. Create `codebases/new-app/codebase.yaml`
2. Create values for **both regions**:
   - `codebases/new-app/values/eu-central-1/*.yaml`
   - `codebases/new-app/values/eu-south-1/*.yaml`
3. Commit and push
4. Both ArgoCD instances automatically create Applications

### Adding a New Cluster to a Region

1. Create `regions/{region}/clusters/{new-cluster}/config.yaml`
2. Add values for all codebases:
   - `codebases/*/values/{region}/{new-cluster}.yaml`
3. Commit and push
4. Regional ArgoCD creates Applications

### Adding a New Region

1. Create `regions/{new-region}/clusters/` structure
2. Create bootstrap: `bootstrap/{new-region}.yaml`
3. Add values for all codebases: `codebases/*/values/{new-region}/`
4. Deploy ArgoCD in new region
5. Apply bootstrap file

## Benefits

1. **Regional Isolation**: Failures in one region don't affect the other
2. **Independent Operations**: Each region can be managed separately
3. **Shared Definitions**: No duplication of codebase configurations
4. **Flexible Configuration**: Regional and cluster-specific overrides
5. **Scalable**: Easy to add more regions or clusters
6. **Clear Organization**: Explicit region boundaries in directory structure

## Considerations

### Naming Collisions

✓ **Solved**: Region prefix in cluster names ensures unique Application names

### Values Complexity

⚠️ **Consideration**: Two-level values hierarchy (region/cluster) adds complexity
✓ **Mitigation**: Clear documentation and examples

### Bootstrap Management

⚠️ **Consideration**: Multiple bootstrap files to maintain
✓ **Mitigation**: Similar structure, only path differences

### Cross-Region Dependencies

⚠️ **Consideration**: Applications can't easily reference resources in other regions
✓ **Design**: Applications should be region-independent where possible

## Migration Path

From current single-region structure to multi-region:

1. Create `regions/` directory structure
2. Move `clusters/` to `regions/eu-central-1/clusters/`
3. Restructure values: `codebases/*/values/*.yaml` → `codebases/*/values/eu-central-1/*.yaml`
4. Update bootstrap to region-specific files
5. Update ApplicationSet to use new paths
6. Test with eu-central-1 first
7. Add eu-south-1 structure
8. Deploy ArgoCD in eu-south-1
9. Apply eu-south-1 bootstrap

## Alternative Approaches Considered

### Alternative 1: Single Bootstrap with Region Parameter

**Approach**: Use Helm to template bootstrap with region parameter

**Pros**: Single bootstrap template
**Cons**: Requires Helm rendering, adds complexity, less explicit

**Decision**: ❌ Rejected - Separate files are clearer

### Alternative 2: Flat Cluster Structure with Region Field

**Approach**: Keep `clusters/` flat, use region field in config

**Pros**: Simpler directory structure
**Cons**: Both ArgoCD instances see all clusters, harder to filter, no clear boundaries

**Decision**: ❌ Rejected - Explicit separation is better

### Alternative 3: Separate Repositories per Region

**Approach**: Complete separate repos for each region

**Pros**: Complete isolation
**Cons**: Duplicates codebase definitions, harder to maintain consistency

**Decision**: ❌ Rejected - Violates DRY principle

## Recommended Implementation

✅ **Chosen Approach**: Region-based directory structure with:
- Separate bootstrap files per region
- Clusters organized under `regions/{region}/`
- Shared codebases with region/cluster values hierarchy
- Region-prefixed cluster names for uniqueness
