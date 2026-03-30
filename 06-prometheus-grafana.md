# Prometheus & Grafana — Monitoring Deep Dive (6 YOE)

---

## Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS STACK                                 │
│                                                                    │
│  Targets             Prometheus          Alertmanager   Grafana    │
│  (apps/K8s) ──scrape──► (TSDB)  ──fire──► (route/     ◄──query──  │
│  /metrics              (store)            notify)     (visualize)  │
│                           │                  │              │      │
│                           │             PagerDuty       Dashboards │
│                           │             Slack           Alerts     │
│                           └──────────────────────────────────────  │
└────────────────────────────────────────────────────────────────────┘
```

- **Prometheus**: Pull-based metrics collection + time series DB + alerting rules
- **Alertmanager**: Routes alerts, deduplicates, silences, groups, notifies
- **Grafana**: Visualization layer. Queries Prometheus (and other data sources) for dashboards

---

## Prometheus Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      PROMETHEUS SERVER                      │
│                                                            │
│  ┌────────────┐   ┌───────────────┐   ┌───────────────┐   │
│  │ Retrieval  │   │    TSDB       │   │  HTTP Server  │   │
│  │ (scrapes   │   │  (local disk  │   │  /metrics     │   │
│  │  targets)  │   │  ~15 days)    │   │  /api/v1/     │   │
│  └────────────┘   └───────────────┘   │  query        │   │
│         │                 │           └───────────────┘   │
│  ┌──────▼──────┐  ┌───────▼──────┐                        │
│  │  Service    │  │  Rule Engine │ ─── fires ──► Alertmgr │
│  │  Discovery  │  │  (alert +    │                        │
│  │  (K8s, etc) │  │   recording) │                        │
│  └─────────────┘  └──────────────┘                        │
└────────────────────────────────────────────────────────────┘

Long-term storage: Thanos / Cortex / VictoriaMetrics (remote_write)
```

---

## Metric Types

### Counter
Always increases. Resets on restart. Use `rate()` / `increase()` to get per-second rate.
```
http_requests_total{method="GET", status="200"} 14523
```

### Gauge
Can go up or down. Current snapshot.
```
memory_usage_bytes 1234567890
active_connections 42
```

### Histogram
Samples observations in configurable buckets. Used for latency distribution.
```
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 850
http_request_duration_seconds_bucket{le="1.0"} 1200
http_request_duration_seconds_bucket{le="+Inf"} 1250
http_request_duration_seconds_sum 542.5
http_request_duration_seconds_count 1250
```

### Summary
Like histogram but calculates quantiles on client side (can't aggregate across instances).
```
rpc_duration_seconds{quantile="0.5"} 0.032
rpc_duration_seconds{quantile="0.99"} 0.512
```

---

## PromQL — Essential Queries

```promql
# ── Instant queries ───────────────────────────────────────────
# Current HTTP request rate (per second, averaged over 5min window)
rate(http_requests_total[5m])

# Error rate by service
rate(http_requests_total{status=~"5.."}[5m])

# Error ratio (errors / total)
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# 99th percentile latency (histogram)
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m])
)

# Memory usage in MB
process_resident_memory_bytes / 1024 / 1024

# CPU usage per pod (GKE)
rate(container_cpu_usage_seconds_total{namespace="production",container!=""}[5m])

# ── Aggregation ───────────────────────────────────────────────
# Total RPS across all pods of a deployment
sum(rate(http_requests_total{app="myapp"}[5m])) by (app)

# RPS by status code
sum(rate(http_requests_total{app="myapp"}[5m])) by (status)

# Error rate per service, top 10
topk(10,
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
)

# ── Kubernetes metrics ────────────────────────────────────────
# Pod CPU usage (cores)
sum(rate(container_cpu_usage_seconds_total{namespace="production",container!="POD",container!=""}[5m]))
  by (pod)

# Pod memory usage
sum(container_memory_working_set_bytes{namespace="production",container!="POD",container!=""})
  by (pod)

# OOMKilled events
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# Pod restart count
increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])

# Node memory pressure
kube_node_status_condition{condition="MemoryPressure",status="true"} == 1

# HPA current vs desired replicas
kube_horizontalpodautoscaler_status_current_replicas
kube_horizontalpodautoscaler_spec_max_replicas

# ── SLO queries ───────────────────────────────────────────────
# Availability SLO: % of successful requests over 30 days
1 - (
  sum(increase(http_requests_total{status=~"5.."}[30d]))
  /
  sum(increase(http_requests_total[30d]))
)

# Latency SLO: % of requests < 200ms
sum(increase(http_request_duration_seconds_bucket{le="0.2"}[30d]))
/
sum(increase(http_request_duration_seconds_count[30d]))
```

---

## Alerting Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: myapp.availability
    interval: 30s
    rules:

    # Alert: High error rate
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{app="myapp",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{app="myapp"}[5m]))
        ) > 0.01
      for: 5m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "High error rate on {{ $labels.app }}"
        description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.app }}"
        runbook: "https://wiki.example.com/runbooks/high-error-rate"

    # Alert: High latency
    - alert: HighLatencyP99
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket{app="myapp"}[5m])) by (le)
        ) > 1.0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "P99 latency > 1s for myapp"
        description: "P99 latency is {{ $value }}s"

    # Alert: Pod crash looping
    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 5
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"

    # Alert: HPA at max replicas (scaling ceiling hit)
    - alert: HPAAtMaxReplicas
      expr: |
        kube_horizontalpodautoscaler_status_current_replicas
          ==
        kube_horizontalpodautoscaler_spec_max_replicas
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} at max replicas"
        description: "May need to increase maxReplicas or investigate load"

  - name: slo.burn-rate
    rules:
    # Fast burn: 14.4x burn rate over 1h (will exhaust 30-day budget in 2h)
    - alert: SLOBurnRateFast
      expr: |
        (
          sum(rate(http_requests_total{status=~"5.."}[1h]))
          /
          sum(rate(http_requests_total[1h]))
        ) > (14.4 * 0.001)   # error budget is 0.1% (99.9% SLO)
      for: 2m
      labels:
        severity: critical
        slo: availability
```

---

## Alertmanager Configuration

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  group_by: ['alertname', 'namespace', 'service']
  group_wait: 30s        # wait before sending first notification
  group_interval: 5m     # wait before sending updates
  repeat_interval: 4h    # re-alert if still firing
  receiver: 'default'

  routes:
  # Critical alerts → PagerDuty (wake someone up)
  - match:
      severity: critical
    receiver: pagerduty
    continue: true        # also send to Slack

  # SRE-owned alerts
  - match:
      team: sre
    receiver: sre-slack

  # Business hours only alerts
  - match:
      severity: warning
    receiver: slack-warnings
    active_time_intervals:
    - business_hours

receivers:
- name: 'default'
  slack_configs:
  - channel: '#alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

- name: 'pagerduty'
  pagerduty_configs:
  - service_key: '<pd-service-key>'
    description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

- name: 'sre-slack'
  slack_configs:
  - channel: '#sre-alerts'
    send_resolved: true

inhibit_rules:
# Suppress warning if critical is already firing for same service
- source_match:
    severity: critical
  target_match:
    severity: warning
  equal: ['alertname', 'namespace', 'service']

time_intervals:
- name: business_hours
  time_intervals:
  - weekdays: ['monday:friday']
    times:
    - start_time: '09:00'
      end_time: '18:00'
```

---

## Prometheus Operator / kube-prometheus-stack

```bash
# Install full monitoring stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values.yaml

# Installs: Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics
```

**ServiceMonitor** — tell Prometheus to scrape your app:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: myapp
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

---

## Grafana — Dashboards

### Key Dashboards to Know

```
1. USE Method (Utilization, Saturation, Errors) — for resources
   - CPU: utilization %, throttle %, saturation (run queue length)
   - Memory: usage %, OOM events
   - Disk: usage %, I/O utilization, error rate
   - Network: bandwidth in/out, packet drops, errors

2. RED Method (Rate, Errors, Duration) — for services
   - Request rate (RPS)
   - Error rate (%)
   - Duration (P50, P95, P99 latency)

3. The Four Golden Signals (Google SRE)
   - Latency (how long requests take, separate successful vs failed)
   - Traffic (demand: RPS, QPS)
   - Errors (rate of failed requests)
   - Saturation (fullness: CPU/memory/queue depth)
```

### Grafana Dashboard as Code

```json
{
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{app=\"myapp\"}[5m])) by (status)",
          "legendFormat": "{{ status }}"
        }
      ]
    }
  ]
}
```

Manage dashboards as code with **Grafonnet** (Jsonnet library) or **Grafana Terraform provider**.

---

## Long-term Storage: Thanos

```
Prometheus (2 weeks local) ──remote_write──► Thanos Sidecar
                                                    │
                                             ┌──────▼──────┐
                                             │  GCS Bucket  │  (unlimited retention)
                                             └──────┬──────┘
                                                    │
                              Thanos Query ◄─────────┘
                                    │
                               Grafana queries
```

---

## Interview Questions — Prometheus/Grafana (6 YOE Level)

**Q: What is the difference between a Counter and a Gauge?**
> **Counter**: Monotonically increasing, only goes up, resets on restart. Use for total counts (requests, errors). Always query with `rate()` or `increase()`. **Gauge**: Can go up or down. Use for current values (memory usage, queue depth, active connections). Query directly.

**Q: How do you implement an SLO alert?**
> Use multi-window multi-burn-rate alerting. Calculate error budget (1 - SLO target). Alert when burn rate exceeds threshold in a time window. Example: 99.9% SLO → 0.1% error budget. Fire critical alert if burning budget at 14.4x in 1h window (will exhaust in 2h). Fire warning if at 6x over 6h window.

**Q: Explain the difference between `rate()` and `irate()`.**
> `rate()`: Average rate over the full range window. Smooth, good for alerting and dashboards. `irate()`: Rate using last two data points only. Captures spikes better but noisy. Use `irate()` for debugging spikes, `rate()` for alerting.

**Q: What is a recording rule and why use it?**
> Pre-computed PromQL expressions stored as new time series. Reduces query time for complex or expensive queries. Essential for dashboards queried frequently. Example: `record: job:http_requests:rate5m` with `expr: sum(rate(http_requests_total[5m])) by (job)`.

**Q: How do you handle Prometheus scalability for 1000+ pods?**
> Use Prometheus Operator with sharding. Use federated Prometheus (global pulls from regional). Use Thanos or Cortex for horizontal scaling and long-term storage. Use recording rules to reduce cardinality. Avoid high-cardinality labels (don't use user_id as a label).
