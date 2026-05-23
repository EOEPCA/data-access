# Horizontal Pod Autoscaling

HPA automatically scales pods based on CPU, memory, or request-rate metrics.

## Prerequisites

**Metrics Server** (for CPU/memory scaling):
```bash
kubectl get deployment metrics-server -n kube-system
```

**Prometheus + Adapter** (for request-rate scaling):
```yaml
monitoring:
  metricsServer:
    enabled: true
  prometheus:
    enabled: true
  prometheusAdapter:
    enabled: true
```

## Configuration

Enable in `values-eoapi.yaml`:

```yaml
stac:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    type: "requestRate"  # "cpu", "requestRate", or "both"
    behavior:
      scaleDown:
        stabilizationWindowSeconds: 300
      scaleUp:
        stabilizationWindowSeconds: 60
    targets:
      cpu: 70              # percentage
      requestRate: 100000m # 100 req/s
  settings:
    resources:
      requests:
        cpu: "512m"
        memory: "1024Mi"
      limits:
        cpu: "1280m"
        memory: "1536Mi"
```

## Recommended Settings

### STAC
```yaml
minReplicas: 2
maxReplicas: 10
type: "requestRate"
targets: {requestRate: 100000m, cpu: 70}
```

### Raster
```yaml
minReplicas: 1
maxReplicas: 8
type: "both"
targets: {requestRate: 50000m, cpu: 75}
```

### Vector
```yaml
minReplicas: 1
maxReplicas: 4
type: "cpu"
targets: {cpu: 70}
```

### Multidim
```yaml
minReplicas: 1
maxReplicas: 4
type: "cpu"
targets: {cpu: 70}
```

### STAC Auth Proxy
```yaml
stac-auth-proxy:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 60
```

## Monitoring

```bash
# Check status
kubectl get hpa -n data-access
kubectl describe hpa eoapi-stac -n data-access

# View metrics
kubectl top pods -n data-access

# Watch scaling
kubectl get hpa -n data-access -w
```

## Key Considerations

**Database connections:** Each pod uses a connection pool. Monitor with:
```bash
kubectl exec -n data-access -l postgres-operator.crunchydata.com/role=master -- \
  psql -U postgres -d eoapi -c "SELECT count(*) FROM pg_stat_activity WHERE datname='eoapi';"
```

**PodDisruptionBudget:** Add for `minReplicas > 1`:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: eoapi-stac
  namespace: data-access
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: eoapi
      app.kubernetes.io/component: stac
```

**Resource requests:** Set to ~70% of observed usage under load.
