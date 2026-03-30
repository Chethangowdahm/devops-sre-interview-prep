# Datadog — Observability Platform Deep Dive (6 YOE)

---

## What is Datadog?

Datadog is a **SaaS observability platform** covering:
- **Infrastructure Monitoring** — hosts, containers, K8s, cloud services
- **APM (Application Performance Monitoring)** — distributed traces, service maps
- **Log Management** — centralized logs, parsing, correlation
- **Synthetic Monitoring** — uptime checks, browser tests
- **RUM (Real User Monitoring)** — frontend performance
- **Security Monitoring (CSPM, SIEM)** — threat detection

---

## Datadog Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                                │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Datadog Agent (DaemonSet — 1 pod per node)                   │   │
│  │                                                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │   │
│  │  │  Metrics │  │   Logs   │  │  Traces  │  │   Process    │ │   │
│  │  │ Collector│  │ Collector│  │  (APM)   │  │   Collector  │ │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Datadog Cluster Agent (1 deployment)                         │   │
│  │  - Kubernetes state metrics                                   │   │
│  │  - Cluster-level checks                                       │   │
│  │  - Autoscaling (via External Metrics API)                     │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  App Pods (with dd-agent sidecar or via DaemonSet)            │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐             │   │
│  │  │  App Container      │  │  App Container      │             │   │
│  │  │  DD_TRACE_* envvars │  │  DD_TRACE_* envvars │             │   │
│  │  │  (APM auto-instr.)  │  │  (APM auto-instr.)  │             │   │
│  │  └─────────────────────┘  └─────────────────────┘             │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
         │ metrics, logs, traces sent to:
         ▼
┌──────────────────────────────────┐
│  DATADOG CLOUD (app.datadoghq.com)│
│  Dashboards │ Monitors │ APM      │
│  Log Explorer │ Alerts │ SLOs     │
└──────────────────────────────────┘
```

---

## Datadog Agent Installation on GKE

```bash
# Install via Helm
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  --namespace monitoring \
  --create-namespace \
  -f datadog-values.yaml
```

```yaml
# datadog-values.yaml
datadog:
  apiKey: <your-api-key>           # from GCP Secret Manager in prod
  appKey: <your-app-key>

  # Kubernetes integration
  kubelet:
    tlsVerify: false
  kubeStateMetricsEnabled: true

  # APM (distributed tracing)
  apm:
    portEnabled: true

  # Log collection
  logs:
    enabled: true
    containerCollectAll: true

  # Process monitoring
  processAgent:
    enabled: true
    processCollection: true

  # Tags for all metrics/logs/traces
  tags:
  - env:production
  - team:sre
  - cluster:wsp-prod

  # Pod annotations auto-discovery
  podAnnotationsAsTags:
    app: service
    version: version

agents:
  tolerations:
  - operator: Exists    # run on all nodes including tainted ones

clusterAgent:
  enabled: true
  metricsProvider:
    enabled: true       # allows K8s HPA to use Datadog metrics
```

---

## APM — Distributed Tracing

### Auto-instrumentation (no code changes)

```yaml
# Add these to your pod spec
spec:
  containers:
  - name: app
    env:
    - name: DD_AGENT_HOST
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
    - name: DD_TRACE_AGENT_PORT
      value: "8126"
    - name: DD_SERVICE
      value: "myapp"
    - name: DD_ENV
      value: "production"
    - name: DD_VERSION
      value: "1.2.0"
    - name: DD_LOGS_INJECTION
      value: "true"          # inject trace_id into logs
    - name: DD_TRACE_SAMPLE_RATE
      value: "1.0"           # 100% sampling (use lower in high-traffic)
    - name: JAVA_TOOL_OPTIONS
      value: "-javaagent:/dd-java-agent.jar"   # Java APM agent
```

### Manual instrumentation (Go example)
```go
import (
    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
    "gopkg.in/DataDog/dd-trace-go.v1/contrib/net/http"
)

func main() {
    tracer.Start(
        tracer.WithServiceName("myapp"),
        tracer.WithEnv("production"),
        tracer.WithServiceVersion("1.2.0"),
    )
    defer tracer.Stop()

    // Instrumented HTTP server
    mux := httptrace.NewServeMux()
    mux.HandleFunc("/api/users", handleUsers)
    http.ListenAndServe(":8080", mux)
}

// Custom spans
func processOrder(ctx context.Context, orderID string) error {
    span, ctx := tracer.StartSpanFromContext(ctx, "processOrder")
    defer span.Finish()

    span.SetTag("order.id", orderID)

    if err := validateOrder(ctx, orderID); err != nil {
        span.SetTag(ext.Error, err)
        return err
    }
    return nil
}
```

### Service Map
APM automatically builds a service dependency map:
```
frontend → api-gateway → user-service → PostgreSQL
                       → payment-service → Stripe API
                       → notification-service → SendGrid
```

---

## Log Management

### Structured Logging (critical for Datadog)
```go
// Use JSON logging — Datadog parses automatically
import "github.com/sirupsen/logrus"

log := logrus.New()
log.SetFormatter(&logrus.JSONFormatter{})

log.WithFields(logrus.Fields{
    "dd.trace_id": span.Context().TraceID(),  // correlate with APM!
    "dd.span_id":  span.Context().SpanID(),
    "user_id":     userID,
    "order_id":    orderID,
    "duration_ms": elapsed.Milliseconds(),
}).Info("Order processed successfully")

// Output:
// {"dd.trace_id":"1234567890","level":"info","msg":"Order processed successfully","time":"..."}
```

### Log Pipeline (parsing)
```
Raw log → Datadog Ingestion → Pipeline (parsing rules) → Structured attributes
          (stream)            Grok patterns               (searchable fields)

Example Grok pattern:
  Rule: access_log
  Pattern: %{IPORHOST:network.client.ip} - %{NOTSPACE:http.auth} \[%{HTTPDATE:date}\] "%{WORD:http.method} %{NOTSPACE:http.url} HTTP/%{NUMBER}" %{NUMBER:http.status_code} %{NUMBER:network.bytes_written}
```

### Log-based Monitors
```
Query: service:myapp status:error
Alert: error count > 100 in 5 minutes
```

---

## Monitors & Alerts

### Monitor Types

| Type | Use case |
|------|---------|
| **Metric** | Alert on Prometheus-style metric thresholds |
| **Log** | Alert on log patterns/counts |
| **APM** | Alert on trace error rate or latency |
| **Synthetic** | Alert if uptime check fails |
| **Anomaly** | ML-based: alert when metric deviates from baseline |
| **Forecast** | Alert before disk/resource exhaustion |
| **Composite** | Combine multiple monitors with AND/OR logic |
| **SLO Alert** | Alert when error budget burn rate is too high |

### Example Monitor (Terraform)
```hcl
resource "datadog_monitor" "high_error_rate" {
  name  = "myapp - High Error Rate"
  type  = "metric alert"

  message = <<-EOT
    Error rate for {{service.name}} is {{value}}%.
    Runbook: https://wiki/runbooks/high-error-rate
    @pagerduty-production @slack-alerts
  EOT

  query = "sum(last_5m):sum:trace.http.request.errors{service:myapp,env:production}.as_rate() / sum:trace.http.request.hits{service:myapp,env:production}.as_rate() * 100 > 1"

  monitor_thresholds {
    critical = 1.0    # > 1% error rate = critical
    warning  = 0.5    # > 0.5% = warning
  }

  notify_no_data    = true
  no_data_timeframe = 10

  tags = ["env:production", "service:myapp", "team:sre"]
}
```

---

## SLOs in Datadog

```
SLO = Service Level Objective
  Target: 99.9% availability over 30 days
  Error budget: 0.1% = 43.8 min/month

Configure in Datadog:
  Type: Metric-based SLO
  Good metric: sum:trace.http.request.hits{service:myapp,env:production,!http.status_code:5*}
  Total metric: sum:trace.http.request.hits{service:myapp,env:production}
  Target: 99.9%
  Time window: 30 days (rolling)

Error budget alerts:
  Burn rate > 14.4x → critical (2h to exhaust budget)
  Burn rate > 1x    → warning (budget will be exhausted by end of window)
```

---

## Dashboards

### Key Dashboards to Build

```
1. Service Health Dashboard (RED method):
   - Request Rate by status code (stacked graph)
   - Error Rate % (target SLO line)
   - P50/P95/P99 Latency (separate graph)
   - Active pods / HPA replicas

2. Infrastructure Dashboard (USE method):
   - CPU utilization per node/pod
   - Memory utilization
   - Pod restarts (heatmap)
   - OOMKilled events

3. SLO Dashboard:
   - Error budget remaining (countdown)
   - Burn rate chart
   - Events (deployments, incidents) overlaid

4. Cost Dashboard:
   - Node costs by pool
   - CPU/memory efficiency (requested vs used)
```

---

## Datadog vs Prometheus/Grafana

| Aspect | Prometheus + Grafana | Datadog |
|--------|---------------------|---------|
| Cost | Free (infra cost only) | Expensive ($$$) |
| Setup | Complex (self-managed) | Easy (SaaS) |
| Retention | 2 weeks (+ Thanos) | 15 months default |
| APM | With Jaeger/Tempo | Built-in, excellent |
| Logs | With Loki | Built-in, excellent |
| Dashboards | Grafana | Datadog Dashboards |
| Alerting | Alertmanager | Datadog Monitors |
| Correlation | Manual (trace_id in logs) | Automatic (unified) |
| Cardinality | Limited | Better (indexed) |
| Scale | Needs Thanos/Cortex | Managed |

**Common pattern**: Use both — Prometheus/Grafana for detailed K8s infra metrics (free), Datadog for APM, logs, and synthetic monitoring.

---

## Interview Questions — Datadog (6 YOE Level)

**Q: How do you correlate logs, metrics, and traces in Datadog?**
> Inject trace and span IDs into logs (`DD_LOGS_INJECTION=true`). Tag everything consistently with `service`, `env`, `version`. In Datadog, you can pivot from a trace to the exact log lines during that trace, and from a log to the trace it belongs to. Add deployment events to dashboards to correlate metric changes with deployments.

**Q: How do you manage Datadog costs at scale?**
> Limit custom metrics (each unique tag combination = a metric). Use log exclusion filters to drop noisy, low-value logs. Reduce APM sampling rate (e.g., 10% for high-traffic services, keep 100% for errors). Archive logs to GCS/S3 for cheaper long-term storage. Use `DD_TRACE_SAMPLE_RATE` with priority sampling.

**Q: How do you set up Datadog for Kubernetes?**
> Install Datadog Agent as DaemonSet + Cluster Agent via Helm. Enable autodiscovery via pod annotations. Use Admission Controller for APM auto-instrumentation (inject library without code changes). Tag pods with `DD_ENV`, `DD_SERVICE`, `DD_VERSION` (Unified Service Tagging) for correlation.

**Q: What is Unified Service Tagging?**
> A standard requiring all telemetry (metrics, logs, traces) from a service to share the same `env`, `service`, `version` tags. Enables jumping from a metric spike → related traces → related logs for the same version of the same service in the same environment. Set via `DD_ENV`, `DD_SERVICE`, `DD_VERSION` env vars.

**Q: How would you monitor a deployment rollout with Datadog?**
> Use Datadog Deployment Tracking (send deploy events via API). Overlay events on dashboards. Set up monitors with alert recovery evaluation window that accounts for the rollout. Use APM to compare P99 latency and error rate between old version and new version. Set up rollback automation: if error rate > X after deploy, trigger Jenkins job to rollback.
