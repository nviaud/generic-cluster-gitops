# Jaeger - Distributed Tracing

Jaeger is an open-source, end-to-end distributed tracing system for monitoring and troubleshooting microservices-based architectures.

## Architecture

### Development
- **All-in-one deployment**: Single pod containing agent, collector, query, and UI
- **Storage**: In-memory (traces lost on restart)
- **Resources**: Minimal (100m CPU, 256Mi memory)
- **Use case**: Local development and testing

### Staging
- **Production-like deployment**: Separate agent, collector, and query components
- **Storage**: Elasticsearch (persistent)
- **Replicas**: 2 for collector and query
- **Use case**: Pre-production testing with realistic architecture

### Production
- **HA deployment**: Separate components with autoscaling
- **Storage**: Elasticsearch with authentication
- **Replicas**: 3 collectors (autoscale 3-10), 2 query instances
- **Monitoring**: Service monitor enabled for Prometheus
- **Use case**: Production workloads

## Components

### Agent
Runs as a sidecar or DaemonSet, receives traces from instrumented applications via UDP/HTTP.

### Collector
Receives traces from agents, validates, indexes, and stores them.

### Query
Provides API and UI for querying and visualizing traces.

### All-in-one
Combined agent, collector, and query in a single binary (dev only).

## Accessing the UI

### Development
```bash
# Port-forward to access UI locally
kubectl port-forward -n tracing svc/jaeger-query 16686:16686

# Open browser
open http://localhost:16686
```

Or access via Ingress:
- URL: https://jaeger-dev.local

### Staging/Production
Access via Ingress:
- Staging: https://jaeger-staging.local
- Production: https://jaeger.local

## Instrumenting Applications

### OpenTelemetry (Recommended)
```go
// Go example
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
)

exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(
    jaeger.WithEndpoint("http://jaeger-collector.tracing.svc.cluster.local:14268/api/traces"),
))
```

### Jaeger Client Libraries
```python
# Python example
from jaeger_client import Config

config = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'local_agent': {
            'reporting_host': 'jaeger-agent.tracing.svc.cluster.local',
            'reporting_port': 6831,
        },
    },
    service_name='my-service',
)
tracer = config.initialize_tracer()
```

## Storage Backends

### Memory (Dev)
- Fast, no dependencies
- Data lost on pod restart
- Limited capacity (5,000 traces)

### Elasticsearch (Staging/Production)
- Persistent storage
- Scalable and searchable
- Requires Elasticsearch cluster

**Setup Elasticsearch:**
```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  -n logging --create-namespace \
  --set replicas=3 \
  --set minimumMasterNodes=2
```

**Configure credentials:**
```bash
# Create secret for Jaeger to access Elasticsearch
kubectl create secret generic jaeger-elasticsearch-credentials \
  -n tracing \
  --from-literal=username=elastic \
  --from-literal=password=YOUR_PASSWORD
```

## Service Endpoints

| Component | Port | Protocol | Description |
|-----------|------|----------|-------------|
| Agent | 6831 | UDP | Compact thrift protocol |
| Agent | 6832 | UDP | Binary thrift protocol |
| Agent | 5778 | HTTP | Serve configs, sampling strategies |
| Collector | 14268 | HTTP | Accept jaeger.thrift over HTTP |
| Collector | 14250 | gRPC | Accept model.proto |
| Query | 16686 | HTTP | Serve frontend and API |

## Sampling Strategies

### Development
100% sampling (all traces captured) for debugging.

### Production
Consider adaptive sampling:
- High-volume services: 1-10% sampling
- Low-volume services: 100% sampling
- Error traces: Always sample

Configure in production values or via remote sampling:
```yaml
collector:
  samplingConfig: |
    {
      "default_strategy": {
        "type": "probabilistic",
        "param": 0.1
      },
      "service_strategies": [
        {
          "service": "critical-service",
          "type": "probabilistic",
          "param": 1.0
        }
      ]
    }
```

## Integration with Grafana

Add Jaeger as a datasource in Grafana:

1. Navigate to Configuration → Data Sources
2. Add Jaeger datasource
3. Configure URL: `http://jaeger-query.tracing.svc.cluster.local:16686`
4. Link traces from Prometheus metrics using trace ID

## Monitoring

Enable ServiceMonitor for Prometheus metrics:
```yaml
# In production.yaml
serviceMonitor:
  enabled: true
```

Key metrics to monitor:
- `jaeger_collector_traces_received_total` - Traces received
- `jaeger_collector_spans_saved_total` - Spans persisted
- `jaeger_collector_spans_rejected_total` - Spans rejected
- `jaeger_query_requests_total` - Query requests

## Troubleshooting

### No traces appearing
1. Check application instrumentation is sending to correct endpoint
2. Verify agent/collector are running: `kubectl get pods -n tracing`
3. Check collector logs: `kubectl logs -n tracing -l app=jaeger-collector`

### UI not accessible
1. Check ingress: `kubectl get ingress -n tracing`
2. Verify query service: `kubectl get svc -n tracing jaeger-query`
3. Port-forward as workaround: `kubectl port-forward -n tracing svc/jaeger-query 16686:16686`

### Elasticsearch connection issues
1. Verify ES is running: `kubectl get pods -n logging`
2. Check credentials secret: `kubectl get secret -n tracing jaeger-elasticsearch-credentials`
3. Test connection from collector pod

### High memory usage
1. Reduce retention period in Elasticsearch
2. Lower sampling rate
3. Increase collector replicas to distribute load
4. Enable autoscaling for collectors

## Best Practices

1. **Use OpenTelemetry** - Future-proof, vendor-neutral instrumentation
2. **Implement adaptive sampling** - Balance cost vs observability
3. **Set retention policies** - Don't keep traces forever
4. **Monitor Jaeger itself** - Use ServiceMonitor and alerts
5. **Secure the UI** - Use authentication/authorization in production
6. **Correlate with metrics** - Link traces to Prometheus/Grafana
7. **Tag appropriately** - Use meaningful tags for filtering

## Related Documentation

- [Jaeger Official Docs](https://www.jaegertracing.io/docs/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Jaeger Helm Chart](https://github.com/jaegertracing/helm-charts)
