# Grafana

This directory contains the configuration for Grafana, the visualization and analytics platform.

## What's Included

- **Grafana**: Visualization platform
- **Pre-configured Datasource**: Prometheus
- **Pre-installed Dashboards**:
  - Kubernetes Cluster Monitoring (GrafanaNet 7249)
  - Node Exporter Full (GrafanaNet 1860)
  - Kubernetes Pods (GrafanaNet 6417)
  - Nginx Ingress Controller (GrafanaNet 9614)
  - ArgoCD (GrafanaNet 14584)

## Default Credentials

**Development Environment:**
- Username: `admin`
- Password: `admin`

**Production Environment:**
- Credentials stored in `grafana-admin-credentials` secret

## Accessing Grafana

### Via Ingress

The default configuration creates an Ingress:
- **Dev**: https://grafana-dev.local

To access locally, add to your hosts file:
```
127.0.0.1 grafana-dev.local
```

Then port-forward the nginx-ingress controller and access via browser.

### Via Port Forward

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Then access at http://localhost:3000

## Pre-configured Dashboards

All dashboards are automatically provisioned:

1. **Kubernetes Cluster** - Overall cluster health and resource usage
2. **Node Exporter** - Detailed node metrics (CPU, memory, disk, network)
3. **Kubernetes Pods** - Pod-level metrics and performance
4. **Nginx Ingress** - Ingress controller metrics and request rates
5. **ArgoCD** - Application sync status and GitOps metrics

## Adding Custom Dashboards

### Option 1: Via UI (Not Recommended for GitOps)

1. Create dashboard in Grafana UI
2. Export as JSON
3. Add to Git (see Option 2)

### Option 2: Via GitOps (Recommended)

Create a ConfigMap in `overlays/<environment>/`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    {
      "dashboard": {
        "title": "My Custom Dashboard",
        "panels": [...]
      }
    }
```

Add to kustomization.yaml and commit.

### Option 3: Import from GrafanaNet

Update the ApplicationSet values in `apps/grafana-appset.yaml`:

```yaml
dashboards:
  default:
    my-new-dashboard:
      gnetId: 12345  # Dashboard ID from grafana.com
      revision: 1
      datasource: Prometheus
```

## Datasources

Prometheus is pre-configured as the default datasource:
- URL: `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090`
- Access: Proxy
- Scrape interval: 30s

### Adding Additional Datasources

Edit the overlay kustomization to add datasources:

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        url: http://loki:3100
        access: proxy
```

## Plugins

Pre-installed plugins:
- grafana-piechart-panel
- grafana-clock-panel

### Adding More Plugins

Update the ApplicationSet values:

```yaml
plugins:
  - grafana-piechart-panel
  - grafana-clock-panel
  - grafana-worldmap-panel  # Add new plugins here
```

## Persistence

Grafana uses a PersistentVolumeClaim:
- Size: 10Gi
- Access: ReadWriteOnce

Dashboard configurations and user preferences are persisted.

## Security

### Change Admin Password

**For Production**, create a secret:

```bash
kubectl create secret generic grafana-admin-credentials \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<strong-password>' \
  -n monitoring
```

### Configure OAuth (Optional)

Edit overlay values to add OAuth providers (GitHub, Google, Azure AD, etc.):

```yaml
env:
  GF_AUTH_GITHUB_ENABLED: "true"
  GF_AUTH_GITHUB_CLIENT_ID: "your-client-id"
  GF_AUTH_GITHUB_CLIENT_SECRET: "your-client-secret"
  GF_AUTH_GITHUB_ALLOWED_ORGANIZATIONS: "your-org"
```

## Troubleshooting

### Grafana Pod Not Starting

Check logs:
```bash
kubectl logs -n monitoring deployment/grafana
```

Common issues:
- Insufficient resources
- PVC not bound
- Invalid configuration

### No Data in Dashboards

1. Check Prometheus datasource is reachable:
   - Go to Configuration → Data Sources
   - Test the Prometheus connection

2. Verify Prometheus is scraping metrics:
   ```bash
   kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
   ```
   Go to http://localhost:9090/targets

### Dashboard Not Loading

Check ConfigMap labels:
```bash
kubectl get configmaps -n monitoring -l grafana_dashboard=1
```

## Useful Grafana Features

### Alerting

Create alerts directly in Grafana:
1. Open a dashboard panel
2. Click Edit → Alert
3. Configure conditions and notifications

### Variables

Use variables for dynamic dashboards:
- Namespace selector
- Pod selector
- Time range

### Annotations

Mark deployments and events on graphs for better correlation.

## Example Queries for Custom Dashboards

### Pod CPU Usage
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="$namespace", pod="$pod"}[5m])) by (container)
```

### Pod Memory Usage
```promql
sum(container_memory_working_set_bytes{namespace="$namespace", pod="$pod"}) by (container)
```

### HTTP Request Rate
```promql
sum(rate(nginx_ingress_controller_requests{namespace="$namespace"}[5m])) by (ingress)
```

### ArgoCD Sync Failures
```promql
sum(increase(argocd_app_sync_total{phase!~"Succeeded|Running"}[1h])) by (name)
```
