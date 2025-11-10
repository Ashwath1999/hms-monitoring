# Monitoring Stack - Hospital Management System

Comprehensive monitoring setup using Prometheus and Grafana for the HMS microservices.

## Components

- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboarding
- **Service Metrics**: All 7 microservices expose `/metrics` endpoint

## Quick Start

### 1. Start Services First

Make sure all HMS services are running:

```bash
cd ../deployment/docker
docker-compose up -d
```

### 2. Start Monitoring Stack

```bash
cd monitoring
docker-compose -f docker-compose.monitoring.yml up -d
```

### 3. Access Dashboards

- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000
  - Username: `admin`
  - Password: `admin123`

## Metrics Available

### HTTP Metrics (All Services)

| Metric Name | Type | Description |
|-------------|------|-------------|
| `http_requests_total` | Counter | Total HTTP requests by method, route, status |
| `http_request_duration_ms` | Histogram | HTTP request duration in milliseconds |

### Business Metrics

**Patient Service:**
- `patients_created_total` - Counter
- `patients_active_total` - Gauge

**Appointment Service:**
- `appointments_created_total` - Counter
- `appointments_cancelled_total` - Counter
- `appointments_rescheduled_total` - Counter

**Billing Service:**
- `bills_created_total` - Counter
- `bill_creation_latency_ms` - Histogram

**Payment Service:**
- `payments_total` - Counter
- `payments_failed_total` - Counter
- `payment_amount_total` - Counter (total amount processed)

**Notification Service:**
- `notifications_sent_total` - Counter (by type)

## Prometheus Queries

### Service Health

```promql
# Up/Down status of all services
up{job=~".*-service"}

# Request rate by service (requests per second)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status_code=~"5.."}[5m])

# Error percentage
100 * (
  rate(http_requests_total{status_code=~"5.."}[5m])
  /
  rate(http_requests_total[5m])
)
```

### Latency Metrics

```promql
# P50 latency by service
histogram_quantile(0.50, rate(http_request_duration_ms_bucket[5m]))

# P90 latency
histogram_quantile(0.90, rate(http_request_duration_ms_bucket[5m]))

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_ms_bucket[5m]))
```

### Business Metrics

```promql
# Appointment creation rate
rate(appointments_created_total[5m])

# Bill creation latency P99
histogram_quantile(0.99, rate(bill_creation_latency_ms_bucket[5m]))

# Payment failure rate
rate(payments_failed_total[5m]) / rate(payments_total[5m])

# Total active patients
patients_active_total
```

## Grafana Dashboards

### Pre-configured Dashboards

1. **HMS System Overview**
   - Request rate by service
   - Error rate by service
   - Response time percentiles (P50/P90/P99)
   - Business metrics (appointments, bills, payments)

### Creating Custom Dashboards

1. Login to Grafana (http://localhost:3000)
2. Click "+" → "Dashboard"
3. Add Panel
4. Select "Prometheus" as data source
5. Enter PromQL query
6. Configure visualization

### Example Dashboard Queries

**Panel: Service Uptime**
```promql
up{job=~".*-service"}
```

**Panel: Top 5 Slowest Endpoints**
```promql
topk(5, histogram_quantile(0.99, sum(rate(http_request_duration_ms_bucket[5m])) by (route, le)))
```

**Panel: Request Volume Heatmap**
```promql
sum(rate(http_requests_total[1m])) by (service)
```

## Alerting (Future Enhancement)

Add alertmanager configuration to `prometheus.yml`:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

Example alert rules:

```yaml
groups:
  - name: service_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_ms_bucket[5m])) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.service }}"
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Grafana :3000                      │
│              (Visualization Layer)                  │
└──────────────────┬──────────────────────────────────┘
                   │ Queries
                   ▼
┌─────────────────────────────────────────────────────┐
│               Prometheus :9090                      │
│             (Metrics Collection)                    │
└──────────┬──────────────────────────────────────────┘
           │ Scrapes /metrics every 15s
           │
           ├─► Patient Service :3001/metrics
           ├─► Doctor Service :3002/metrics
           ├─► Appointment Service :3003/metrics
           ├─► Billing Service :3004/metrics
           ├─► Prescription Service :3005/metrics
           ├─► Payment Service :3006/metrics
           └─► Notification Service :3007/metrics
```

## Troubleshooting

### Prometheus Can't Scrape Services

```bash
# Check if services are running
docker ps | grep hms

# Check Prometheus targets
# Go to http://localhost:9090/targets
# All targets should be "UP"

# Check network connectivity
docker exec hms-prometheus ping patient-service
```

### Grafana Can't Connect to Prometheus

```bash
# Check if both are on same network
docker network inspect hms-network

# Verify Prometheus is accessible
docker exec hms-grafana curl http://prometheus:9090/api/v1/status/config
```

### No Metrics Showing Up

```bash
# Verify metrics endpoint works
curl http://localhost:3001/metrics

# Check Prometheus logs
docker logs hms-prometheus

# Check Grafana logs
docker logs hms-grafana
```

## Kubernetes Deployment

For Kubernetes, use Prometheus Operator or create ConfigMaps:

```yaml
# prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: hms
data:
  prometheus.yml: |
    # Copy from prometheus/prometheus.yml
```

Then deploy with:

```bash
kubectl apply -f monitoring/kubernetes/
```

## Best Practices

1. **Retention**: Configure appropriate retention periods
   ```
   --storage.tsdb.retention.time=30d
   --storage.tsdb.retention.size=10GB
   ```

2. **Recording Rules**: Use recording rules for frequently queried metrics

3. **Labels**: Use consistent labels across all services

4. **Cardinality**: Avoid high-cardinality labels (like user IDs)

5. **Sampling**: For high-traffic services, consider metric sampling

## Stopping Monitoring

```bash
docker-compose -f docker-compose.monitoring.yml down

# To remove data volumes as well
docker-compose -f docker-compose.monitoring.yml down -v
```

## Production Considerations

For production deployment:

1. **High Availability**: Run multiple Prometheus instances
2. **Long-term Storage**: Use Thanos or Cortex
3. **Alerting**: Set up Alertmanager
4. **Authentication**: Enable auth for Grafana
5. **HTTPS**: Use TLS for all endpoints
6. **Backup**: Regular backup of Grafana dashboards and Prometheus data
7. **Resource Limits**: Set appropriate CPU/memory limits

## License

MIT
