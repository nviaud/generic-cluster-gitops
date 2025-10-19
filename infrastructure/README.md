# Infrastructure Directory

This directory contains Helm values and Kustomize overlays for all applications managed by ArgoCD.

## Directory Structure

Each application follows the same consistent pattern:

```
infrastructure/{application}/
├── values/                    # Helm values files
│   ├── common.yaml           # Shared across all environments
│   ├── dev.yaml              # Dev-specific overrides only
│   ├── staging.yaml          # Staging-specific overrides only
│   └── production.yaml       # Production-specific overrides only
├── base/                      # Kustomize base resources
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── ...                    # Other base resources
└── overlays/                  # Environment-specific Kustomize overlays
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Applications

### cert-manager
Certificate management for Kubernetes using Let's Encrypt or self-signed certificates.

**Common values:** CRD installation, leader election settings
**Environment differences:** Minimal (mostly the same across environments)

### external-dns
Automatically configures DNS records based on Ingress resources.

**Common values:** Sources (service/ingress), policy (sync), registry (txt)
**Environment differences:**
- `txtOwnerId`: Unique per environment
- `logLevel`: debug in dev, info in staging/production

### grafana
Metrics visualization and dashboarding.

**Common values:** Dashboards, datasources, plugins, service configuration
**Environment differences:**
- Replicas: 1 in dev, 2 in staging/production
- Resources: Lower in dev, higher in staging/production
- Secrets: Hardcoded in dev, from secret store in staging/production
- Log level: debug in dev, info in staging/production

### metrics-server
Kubernetes metrics API for horizontal pod autoscaling.

**Common values:** Base arguments for metrics collection
**Environment differences:**
- Replicas: 1 in dev, 2 in staging/production
- TLS validation: Insecure in dev, secure in staging/production

### nginx-ingress
Ingress controller for routing external traffic to services.

**Common values:** Service type (LoadBalancer), metrics configuration
**Environment differences:**
- Replicas: 1 in dev, 2 in staging/production
- Resources: Lower in dev, higher in staging/production

### prometheus
Metrics collection and alerting (kube-prometheus-stack).

**Common values:** Exporters, alert rules, service monitors
**Environment differences:**
- Replicas: 1 in dev, 2 in staging/production
- Retention: 7d in dev, 15d in staging, 30d in production
- Storage: 10Gi in dev, 30Gi in staging, 50Gi in production
- Resources: Scaled appropriately per environment

## How Values Are Merged

ArgoCD ApplicationSets reference both common and environment-specific values files:

```yaml
helm:
  valueFiles:
    - $values/infrastructure/{app}/values/common.yaml      # Applied first
    - $values/infrastructure/{app}/values/{{.environment}}.yaml  # Overrides common
```

Helm merges these files from left to right, with later files overriding earlier ones.

**Example:**

```yaml
# common.yaml
replicas: 1
resources:
  requests:
    cpu: 100m
    memory: 128Mi

# production.yaml
replicas: 2              # Overrides common.yaml
resources:
  requests:
    cpu: 500m           # Overrides common.yaml
    memory: 2Gi         # Overrides common.yaml
```

**Result in production:**
```yaml
replicas: 2
resources:
  requests:
    cpu: 500m
    memory: 2Gi
```

## Adding a New Application

1. **Create directory structure:**
   ```bash
   mkdir -p infrastructure/{app-name}/values
   mkdir -p infrastructure/{app-name}/base
   mkdir -p infrastructure/{app-name}/overlays/{dev,staging,production}
   ```

2. **Create value files:**
   - `values/common.yaml` - Configuration shared across all environments
   - `values/dev.yaml` - Dev-specific overrides only
   - `values/staging.yaml` - Staging-specific overrides only
   - `values/production.yaml` - Production-specific overrides only

3. **Create ApplicationSet:**
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ApplicationSet
   metadata:
     name: {app-name}
     namespace: argocd
   spec:
     goTemplate: true
     goTemplateOptions: ["missingkey=error"]
     generators:
       - list:
           elements:
             - environment: dev
             - environment: staging
             - environment: production
     template:
       metadata:
         name: '{app-name}-{{.environment}}'
         namespace: argocd
       spec:
         project: default
         sources:
           - repoURL: https://charts.example.com
             chart: {chart-name}
             targetRevision: {version}
             helm:
               releaseName: {app-name}
               valueFiles:
                 - $values/infrastructure/{app-name}/values/common.yaml
                 - $values/infrastructure/{app-name}/values/{{.environment}}.yaml
           - repoURL: https://github.com/NicolasViaud/cluster-gitops.git
             targetRevision: main
             ref: values
           - repoURL: https://github.com/NicolasViaud/cluster-gitops.git
             targetRevision: main
             path: 'infrastructure/{app-name}/overlays/{{.environment}}'
         destination:
           server: https://kubernetes.default.svc
           namespace: {namespace}
         syncPolicy:
           automated:
             prune: true
             selfHeal: true
   ```

## Best Practices

1. **Keep common.yaml DRY** - Only put truly common config there
2. **Environment files override only** - Don't repeat common values
3. **Use comments** - Explain why specific overrides exist
4. **Version control everything** - Except plain secrets
5. **Test locally first:**
   ```bash
   helm template {app} {chart} \
     -f infrastructure/{app}/values/common.yaml \
     -f infrastructure/{app}/values/dev.yaml
   ```
6. **Compare environments easily:**
   ```bash
   diff infrastructure/{app}/values/dev.yaml \
        infrastructure/{app}/values/production.yaml
   ```

## Secrets Management

See [../SECRETS-MANAGEMENT.md](../SECRETS-MANAGEMENT.md) for detailed information on managing secrets per environment.

## Troubleshooting

**Q: My values aren't being applied**
A: Check that the ApplicationSet has a source with `ref: values` pointing to your Git repo.

**Q: Environment-specific value not overriding common value**
A: Ensure the YAML path is identical in both files. Helm merges by key path.

**Q: How do I see what values are actually applied?**
A: Use ArgoCD UI or CLI to inspect the rendered manifests:
```bash
argocd app manifests {app-name}-dev
```

**Q: Can I test values locally?**
A: Yes, using Helm:
```bash
helm template my-app chart-repo/chart-name \
  -f infrastructure/app/values/common.yaml \
  -f infrastructure/app/values/dev.yaml
```

## Related Documentation

- [../REFACTORING-SUMMARY.md](../REFACTORING-SUMMARY.md) - Why we refactored and what changed
- [../SECRETS-MANAGEMENT.md](../SECRETS-MANAGEMENT.md) - How to manage secrets
- [ArgoCD ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Helm Values Files](https://helm.sh/docs/chart_template_guide/values_files/)
