# Prometheus Stack

This directory contains the configuration for the Prometheus monitoring stack using the `kube-prometheus-stack` Helm chart.

## What's Included

- **Prometheus Operator**: Manages Prometheus instances
- **Prometheus**: Time-series database for metrics
- **Alertmanager**: Handles alerts sent by Prometheus
- **Node Exporter**: Exports hardware and OS metrics
- **Kube State Metrics**: Exports Kubernetes cluster state metrics
- **ServiceMonitors**: Pre-configured for ArgoCD and Nginx Ingress

## Components

### Prometheus
- **Dev**: 1 replica, 7 days retention, 10Gi storage
- **Staging/Prod**: 2 replicas, 30 days retention, 50Gi storage

### Alertmanager
- **Dev**: 1 replica
- **Staging/Prod**: 2 replicas

## Accessing Prometheus

### Port Forward
```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Then access at http://localhost:9090

### Via Ingress (if configured)
If you add an Ingress resource in the overlay, you can access via your configured domain.

## ServiceMonitors

ServiceMonitors tell Prometheus what to scrape:

- **Nginx Ingress**: Scrapes ingress controller metrics
- **ArgoCD**: Scrapes all ArgoCD component metrics

### Adding Custom ServiceMonitors

Create a file in `overlays/<environment>/servicemonitor-<name>.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - my-namespace
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

Then add it to the kustomization.yaml in the overlay.

## Querying Metrics

### Example PromQL Queries

**Pod CPU Usage:**
```promql
sum(rate(container_cpu_usage_seconds_total{pod!=""}[5m])) by (pod)
```

**Pod Memory Usage:**
```promql
sum(container_memory_working_set_bytes{pod!=""}) by (pod)
```

**ArgoCD Application Sync Status:**
```promql
argocd_app_info{sync_status="Synced"}
```

**Nginx Ingress Request Rate:**
```promql
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress)
```

## Alerting

Alertmanager is configured but needs a receiver. To configure Slack/Email/etc:

1. Create a secret with your webhook/credentials
2. Add configuration to the overlay kustomization

Example Alertmanager config for Slack:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
    route:
      receiver: 'slack-notifications'
      group_by: ['alertname', 'cluster', 'service']
    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#alerts'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

## Troubleshooting

### Prometheus Not Scraping Targets

Check ServiceMonitor labels match the Prometheus selector:

```bash
kubectl get servicemonitors -n monitoring
kubectl describe prometheus -n monitoring
```

### High Memory Usage

Reduce retention period or sample rate in the overlay values.

### Missing Metrics

Check if the target service has a metrics endpoint:

```bash
kubectl port-forward -n <namespace> svc/<service> 8080:8080
curl http://localhost:8080/metrics
```
