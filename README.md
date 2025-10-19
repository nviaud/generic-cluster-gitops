# Cluster GitOps

This repository contains the GitOps configuration for managing Kubernetes clusters using ArgoCD. It follows a declarative approach where the desired state of infrastructure and applications is defined in Git.

## Repository Structure

```
.
├── apps/                     # ArgoCD ApplicationSets for infrastructure components
│   ├── cert-manager-appset.yaml
│   ├── external-dns-appset.yaml
│   ├── metrics-server-appset.yaml
│   └── nginx-ingress-appset.yaml
├── bootstrap/                # Root application for bootstrapping ArgoCD
│   ├── kustomization.yaml
│   └── root-app.yaml
└── infrastructure/           # Infrastructure component configurations
    ├── cert-manager/
    ├── external-dns/
    ├── metrics-server/
    └── nginx-ingress/
        ├── base/
        └── overlays/
            └── dev/
```

## Architecture

This GitOps setup uses the **App of Apps** pattern:

1. **Root Application** ([bootstrap/root-app.yaml](bootstrap/root-app.yaml)) - The parent application that manages all ApplicationSets
2. **ApplicationSets** ([apps/](apps/)) - Automatically generate Applications for each environment (dev, staging, production)
3. **Infrastructure Configurations** ([infrastructure/](infrastructure/)) - Kustomize overlays for environment-specific configurations

## Prerequisites

- A Kubernetes cluster (v1.20+)
- `kubectl` configured to access your cluster
- `git` installed locally

## Installation

### Step 1: Install ArgoCD

Install ArgoCD in your Kubernetes cluster:

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD components to be ready:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Step 2: Access ArgoCD UI (Optional)

If you want to access the ArgoCD web UI:

```bash
# Port-forward to the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then access the UI at https://localhost:8080

To get the initial admin password:

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Username: `admin`

### Step 3: Bootstrap the Root Application

Apply the root application to bootstrap all infrastructure components:

```bash
# Apply the bootstrap configuration
kubectl apply -k bootstrap/
```

This will create the root ArgoCD Application which will automatically:
- Discover all ApplicationSets in the [apps/](apps/) directory
- Deploy infrastructure components to the appropriate environments
- Enable auto-sync, self-heal, and pruning for all applications

Verify the root application was created:

```bash
kubectl get application -n argocd root-app
```

### Step 4: Verify Deployment

Check that all ApplicationSets were created:

```bash
kubectl get applicationsets -n argocd
```

Check the generated Applications:

```bash
kubectl get applications -n argocd
```

You should see applications for:
- `nginx-ingress-dev`
- `cert-manager-dev`
- `external-dns-dev`
- `metrics-server-dev`

Monitor the sync status:

```bash
# Watch all applications
kubectl get applications -n argocd -w

# Or check sync status
kubectl get applications -n argocd -o wide
```

## Managing Infrastructure

### Adding New Environments

To enable staging or production environments, edit the ApplicationSet files in [apps/](apps/):

```yaml
generators:
  - list:
      elements:
        - environment: dev
        - environment: staging      # Uncomment
        - environment: production   # Uncomment
```

Then ensure corresponding overlays exist in `infrastructure/<component>/overlays/<environment>/`.

### Adding New Infrastructure Components

1. Create the infrastructure configuration:
   ```
   infrastructure/<component>/
   ├── base/
   │   ├── kustomization.yaml
   │   └── ... (base resources)
   └── overlays/
       └── dev/
           └── kustomization.yaml
   ```

2. Create an ApplicationSet in `apps/<component>-appset.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ApplicationSet
   metadata:
     name: <component>
     namespace: argocd
   spec:
     generators:
       - list:
           elements:
             - environment: dev
     template:
       metadata:
         name: '<component>-{{.environment}}'
       spec:
         source:
           repoURL: https://github.com/NicolasViaud/cluster-gitops.git
           targetRevision: main
           path: 'infrastructure/<component>/overlays/{{.environment}}'
         destination:
           server: https://kubernetes.default.svc
           namespace: <component>
         syncPolicy:
           automated:
             prune: true
             selfHeal: true
   ```

3. Commit and push - ArgoCD will automatically detect and deploy the new component.

### Modifying Configurations

1. Edit the infrastructure configuration files in the appropriate overlay directory
2. Commit and push changes to the repository
3. ArgoCD will automatically detect and sync the changes (due to `automated` sync policy)

## Sync Policies

All applications are configured with:

- **Automated Sync**: Changes are automatically applied when detected in Git
- **Self-Heal**: ArgoCD will revert manual changes to match Git state
- **Prune**: Resources deleted from Git will be removed from the cluster
- **CreateNamespace**: Namespaces are created automatically if they don't exist

## Troubleshooting

### View Application Status

```bash
# List all applications
kubectl get applications -n argocd

# Describe a specific application
kubectl describe application <app-name> -n argocd

# View application logs
kubectl logs -n argocd deployment/argocd-application-controller
```

### Sync Issues

If an application is out of sync:

```bash
# Manual sync (if auto-sync is disabled)
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'
```

Or use the ArgoCD CLI:

```bash
argocd app sync <app-name>
```

### Force Refresh

```bash
# Force ArgoCD to refresh application state
kubectl patch application <app-name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

## ArgoCD CLI (Optional)

Install the ArgoCD CLI for easier management:

```bash
# Linux/macOS
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Windows (using WSL or Git Bash)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-windows-amd64.exe
```

Login to ArgoCD:

```bash
argocd login localhost:8080
```

## Uninstalling

To remove all managed infrastructure and ArgoCD:

```bash
# Delete the root application (this will cascade delete all managed apps)
kubectl delete -k bootstrap/

# Delete ArgoCD itself
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Delete the namespace
kubectl delete namespace argocd
```

## Repository Information

- **Repository**: https://github.com/NicolasViaud/cluster-gitops.git
- **Branch**: main
- **Pattern**: App of Apps with ApplicationSets

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Kustomize Documentation](https://kustomize.io/)
