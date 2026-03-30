# Monitoring & Observability - Prometheus, Grafana, Datadog
> Senior SRE Level | Observability Expert

---

## THE THREE PILLARS OF OBSERVABILITY

| Pillar | What | Examples |
|---|---|---|
| **Metrics** | Numeric measurements over time | CPU%, error rate, latency p99 |
| **Logs** | Discrete event records | Access logs, error logs, audit logs |
| **Traces** | Request journey across services | Jaeger, Zipkin, Datadog APM |

---

## PROMETHEUS

### What is Prometheus?
Prometheus is an open-source monitoring and alerting system. Pull-based metrics collection. Time-series database. Uses PromQL query language.

### Architecture
Prometheus pulls /metrics endpoints every ~15s. Each target exposes metrics in Prometheus text format.
Alertmanager handles routing alerts to Slack, PagerDuty, email.
Grafana queries Prometheus via PromQL for dashboards.

### PromQL Essentials
```promql
# CPU usage per pod
sum(rate(container_cpu_usage_seconds_total{namespace='production'}[5m])) by (pod)

# HTTP error rate
sum(rate(http_requests_total{status=~'5..'}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# Request latency p99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Pod restarts in last hour
increase(kube_pod_container_status_restarts_total[1h]) > 5

# Disk space below 10%
node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1
```

### Alerting Rules
```yaml
groups:
- name: production-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~'5..', job='myapp'}[5m]))
      / sum(rate(http_requests_total{job='myapp'}[5m])) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: 'High error rate detected'
      runbook: 'https://wiki/runbooks/high-error-rate'

  - alert: PodCrashLooping
    expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
    for: 0m
    labels:
      severity: warning
```

### ServiceMonitor (Prometheus Operator)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

---

## GRAFANA

### Key Concepts
- **Dashboard**: Collection of panels
- **Panel**: Single visualization (graph, gauge, table)
- **Data Source**: Prometheus, Loki, CloudWatch, Datadog
- **Variable**: Dynamic filters for dashboards

### Dashboard Methodologies
- **USE Method** (for resources): Utilization, Saturation, Errors
- **RED Method** (for services): Rate, Errors, Duration

---

## DATADOG

### What is Datadog?
Commercial SaaS observability platform. Metrics, logs, APM traces, synthetics, profiling in one platform.

### Datadog on Kubernetes
Datadog Agent runs as DaemonSet on every node. Collects host metrics, container metrics, logs, APM traces.

### Key Datadog Features
- Infrastructure metrics and dashboards
- APM distributed tracing (auto-instrumentation)
- Log management (ingestion, parsing, search)
- Synthetic monitoring (uptime, browser tests)
- SLO tracking with error budget
- Anomaly detection

---

## SLI, SLO, SLA, ERROR BUDGETS

| Term | Definition | Example |
|---|---|---|
| **SLI** | Measurable metric | 99.5% requests succeed |
| **SLO** | Target for SLI | 99.9% availability |
| **SLA** | Business contract with penalties | 99.5% or credits |
| **Error Budget** | Allowed unreliability | 43.2 min/month at 99.9% |

**Error budget math**: 99.9% SLO = 0.1% budget = 43.2 min/month downtime allowed.

When error budget exhausted: Freeze new features, focus on reliability.

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: How does Prometheus scraping work?
Prometheus pulls /metrics from targets at configured interval (default 15s). Uses service discovery (K8s API, Consul) to find targets. Pushgateway for short-lived jobs.

**Why pull vs push**: Pull gives Prometheus control over load. Easy to detect down targets.

### Q2: Counter vs Gauge vs Histogram vs Summary?

| Type | Behavior | Use Case |
|---|---|---|
| Counter | Monotonically increasing | requests_total, errors_total |
| Gauge | Up and down | active_connections, memory_bytes |
| Histogram | Distribution in buckets | request_duration |
| Summary | Pre-calculated quantiles | request_duration (cannot aggregate) |

**Prefer histogram** over summary: Histograms can be aggregated across replicas. Summaries cannot.

### Q3: How do you set up alerting for a new service?
1. Instrument app: Add /metrics with RED metrics (rate, errors, duration)
2. Create ServiceMonitor for Prometheus Operator
3. Write PrometheusRule with alerting rules based on SLOs
4. Configure Alertmanager routing (Slack, PagerDuty)
5. Create Grafana dashboard with RED method
6. Write runbook: what to do when alert fires

### Q4: Explain SLO-based alerting (burn rate)
Alert when error budget burns too fast, not just when threshold exceeded.

99.9% SLO = 0.1% error budget. Monthly = 43.2 min.
If error rate is 1% (10x budget), budget exhausted in 4.3 days.
Alert: error rate > 14x normal = will exhaust budget in 2 days.

Better than fixed thresholds: Alerts are proportional to business impact. Reduces alert fatigue.

### Q5: Datadog vs Prometheus - when to use each?

| Aspect | Prometheus | Datadog |
|---|---|---|
| Cost | Free | Expensive |
| Management | Self-managed | Fully managed |
| APM | Separate (Jaeger/Tempo) | Built-in |
| Logs | Separate (Loki/ELK) | Built-in |
| Scale | Complex ops at scale | Auto-scales |

Prometheus for: cost-conscious, open-source, K8s-native setups.
Datadog for: enterprise, single platform for all observability needs.

### Q6: How do you troubleshoot high latency with tracing?
1. Alert fires: endpoint p99 > 2s
2. Open Datadog APM/Jaeger, filter traces for that endpoint with high latency
3. View flame graph of slow trace
4. Find: 1.8s spent in PostgreSQL query
5. Check slow query log - missing index
6. Add index, latency drops to 50ms

### Q7: Real-world: How did you set up observability at Vendasta?
Talk about:
- Kube Prometheus Stack deployed via Helm for cluster-wide monitoring
- Custom ServiceMonitors for each microservice
- Datadog for APM, log correlation, and SLO tracking
- Grafana dashboards with RED method for each service
- SLO-based alerting with burn rates
- Runbooks linked from alert annotations
- On-call rotation with Alertmanager → PagerDuty integration

---

## PRACTICAL: PromQL Cheat Sheet

```promql
# Rate of change (for counters)
rate(http_requests_total[5m])       # Per-second rate over 5min window
irate(http_requests_total[5m])      # Instant rate (last 2 data points)

# Aggregation
sum(metric) by (pod)                # Sum, group by pod
avg(metric) by (namespace)          # Average by namespace
topk(5, metric)                     # Top 5 values

# Histogram percentile
histogram_quantile(0.99, rate(request_duration_bucket[5m]))

# Predict disk full
predict_linear(node_filesystem_avail_bytes[1h], 4*3600) < 0
# When will disk be full in 4 hours?

# Compare current vs 1 week ago
metric offset 1w
```
