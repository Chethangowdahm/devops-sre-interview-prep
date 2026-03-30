# Complete Interview Session — File 7: Monitoring & Observability Round

> **Round Type:** Monitoring deep dive — Prometheus, Grafana, Datadog, SLOs, alerting
> **Your Edge:** You own observability at Vendasta using Datadog APM, Prometheus, Grafana on GKE.

---

## SECTION 1: PROMETHEUS & GRAFANA

### Q1. How does Prometheus work? Explain the pull model.

**[YOUR ANSWER]:**
Prometheus uses a pull model — it scrapes metrics from HTTP endpoints (/metrics) on a configured interval, unlike push-based systems where agents send data to a central server.

**Architecture:**
- Prometheus server: scrapes and stores metrics in TSDB (time-series database)
- Exporters: expose metrics from systems that don't natively do so (node-exporter for Linux, postgres-exporter for Postgres)
- Alertmanager: receives alerts from Prometheus, deduplicates, routes to Slack/PagerDuty/email
- Pushgateway: for short-lived jobs that can't be scraped (batch jobs push metrics before exiting)

**In Kubernetes:**
ServiceMonitor CRD (from Prometheus Operator) tells Prometheus which services to scrape:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-api-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: payment-api
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

**At Vendasta:** We use Prometheus Operator on GKE. Every microservice exposes /metrics endpoint. ServiceMonitors define what to scrape. All metrics flow into Grafana dashboards for visualization.

---

### Q2. Write a PromQL query to find HTTP error rate for a service.

**[YOUR ANSWER — Real PromQL examples]:**

```promql
# Error rate (5xx) as a percentage over last 5 minutes
sum(rate(http_requests_total{service="payment-api", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="payment-api"}[5m])) * 100
```

```promql
# p99 latency for a specific endpoint
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="payment-api",path="/checkout"}[5m]))
  by (le)
)
```

```promql
# Number of pods not in Running state per namespace
kube_pod_status_phase{phase!="Running"} == 1
```

```promql
# Memory usage as % of limit
container_memory_working_set_bytes{container!=""}
/
on(pod, container, namespace) kube_pod_container_resource_limits{resource="memory"} * 100
```

```promql
# Alert: Error budget burn rate too high (multi-window multi-burn-rate)
sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h])) > 0.001
```

**At Vendasta:** I write PromQL for SLO burn rate alerts — if error budget is burning 14x faster than sustainable, wake someone up immediately.

---

### Q3. Explain the RED and USE methods for monitoring.

**[YOUR ANSWER]:**

**RED Method (for services/microservices):**
- **R**ate: requests per second — how much traffic?
- **E**rrors: failed requests per second — what is failing?
- **D**uration: latency distribution (p50, p95, p99) — how slow?

RED is ideal for user-facing services. Ask: 'Is my API healthy for users?'

**USE Method (for infrastructure/resources):**
- **U**tilization: % of time resource is busy (CPU usage, disk bandwidth)
- **S**aturation: amount of work queued/waiting (CPU run queue, disk I/O queue)
- **E**rrors: error count (disk errors, network packet drops)

USE is ideal for infrastructure. Ask: 'Is my server healthy?'

**At Vendasta:** I built Grafana dashboards using both methods. RED dashboards per microservice for development teams, USE dashboards per GKE node for platform team. This split ownership improves incident response — each team knows exactly which dashboard to look at.

---

### Q4. How do you design an alerting strategy that reduces alert fatigue?

**[YOUR ANSWER]:**
Alert fatigue happens when too many low-signal alerts desensitize engineers. The fix is to alert on symptoms, not causes.

**Principles I follow:**
1. Alert on user impact, not internal metrics
   - Bad alert: 'CPU > 80%' — CPU can be high without affecting users
   - Good alert: 'p99 latency > 1s for 5 minutes' — this directly impacts users
2. Every alert must be actionable — if you can't do anything about it, remove the alert
3. Use multi-window multi-burn-rate for SLO-based alerting (fast burn + slow burn)
4. Distinguish between 'page now' (P1, wake me up) and 'check next day' (warning tickets)
5. Regularly audit firing alerts — if an alert fires and nobody acts on it, it should be deleted or reclassified

**Alert template in Prometheus:**
```yaml
groups:
- name: payment-api-slo
  rules:
  - alert: PaymentAPIErrorBudgetCritical
    expr: |
      (
        sum(rate(http_requests_total{service="payment-api",status=~"5.."}[1h]))
        /
        sum(rate(http_requests_total{service="payment-api"}[1h]))
      ) > 0.001
    for: 2m
    labels:
      severity: critical
      team: payments
    annotations:
      summary: Payment API error budget burning at critical rate
      runbook: https://wiki.vendasta.com/runbooks/payment-api-errors
      description: Error rate is {{ $value | humanizePercentage }} over last 1 hour
```

**At Vendasta:** I reduced our weekly alert count from 150+ to under 20 by auditing every alert rule, removing 60% of noise alerts, and converting infrastructure alerts to SLO-based burn rate alerts. On-call engineer satisfaction improved significantly.

---

## SECTION 2: DATADOG

### Q5. How do you use Datadog for APM (Application Performance Monitoring)?

**[YOUR ANSWER]:**
Datadog APM provides distributed tracing — you can see the full request journey from frontend to backend to database.

**Setup in Kubernetes:**
1. Deploy Datadog Agent as DaemonSet (one agent per node)
2. Add APM library to application (dd-trace-go for Go, dd-trace-py for Python, etc.)
3. Set environment variables:
```yaml
env:
- name: DD_ENV
  value: production
- name: DD_SERVICE
  value: payment-api
- name: DD_VERSION
  valueFrom:
    fieldRef:
      fieldPath: metadata.labels['app.kubernetes.io/version']
- name: DD_TRACE_AGENT_HOSTNAME
  valueFrom:
    fieldRef:
      fieldPath: status.hostIP  # Send to node-local agent
```

**Key APM features I use:**
- Flame graphs: visualize where time is spent in a request
- Service Map: see dependencies between all services automatically
- Error tracking: group similar errors with stack traces
- Database monitoring: slow queries, missing indexes

**At Vendasta:** Datadog APM reduced our MTTD (mean-time-to-diagnose) from 25 minutes to under 5 minutes. During incidents, we go straight to the flame graph and see exactly which downstream call is slow or failing.

---

### Q6. How do you define and track SLOs in Datadog?

**[YOUR ANSWER]:**
Datadog has a native SLO feature that tracks compliance against defined targets.

**Types of SLOs in Datadog:**
- **Metric-based SLO:** Ratio of good events to total events
  - Example: (successful requests / total requests) >= 99.9%
- **Monitor-based SLO:** How much time is a monitor in OK state
  - Less precise but easier to set up

**Metric-based SLO definition:**
```
Good events: sum:trace.web.request.hits{env:production,service:payment-api,http.status_code:200}.as_count()
Total events: sum:trace.web.request.hits{env:production,service:payment-api}.as_count()
Target: 99.9% over rolling 30 days
```

**Error Budget Burn Rate alert in Datadog:**
I set up burn rate monitors — if we're consuming error budget 14x faster than the sustainable rate over 1 hour, it's a critical alert. If burning 3x over 6 hours, it's a warning. This follows the Google SRE multi-window alerting pattern.

**At Vendasta:** We have SLOs defined for all Tier-1 services in Datadog. Monthly SLO report goes to engineering leadership automatically. When error budget is below 50% for the month, feature deployments are frozen for that service.

---

### Q7. Scenario: Your monitoring shows high latency but logs show no errors. How do you investigate?

**[YOUR ANSWER]:**
High latency with no errors is a classic scenario — usually a saturation or downstream dependency issue.

**Investigation steps:**
1. **Check Datadog APM traces** — flame graph will show where the time is going
   - Is it DB queries? Redis? External API calls? Application code?
2. **Check infrastructure metrics:**
   - CPU: Is the service CPU-throttled? Check `container_cpu_cfs_throttled_seconds_total`
   - Memory: Near memory limit? GC pauses cause latency spikes
   - Network: Are there retries happening?
3. **Check downstream services:**
   - Database: EXPLAIN ANALYZE slow queries
   - Redis: `SLOWLOG GET 10`
   - External APIs: Third-party status page
4. **Check Kubernetes:**
   - Is HPA at max replicas? Service overloaded
   - Any node pressure causing pod evictions?
   - DNS resolution latency? (`kubectl exec -it ... -- time nslookup service-name`)
5. **Correlate with deployments:**
   - Was there a recent deploy? New code may have introduced a slow operation

**Real example at Vendasta:** High latency with no errors turned out to be CPU throttling. The pod's CPU limit was 500m but the service needed 2 CPU during business hours. `container_cpu_cfs_throttled_seconds_total` in Prometheus showed 70% throttling. We adjusted limits and latency returned to normal immediately.

---

## SECTION 3: SLI/SLO/ERROR BUDGET

### Q8. What SLIs would you define for an e-commerce checkout service?

**[YOUR ANSWER]:**
For a checkout service, I would define SLIs across three categories:

**Availability SLI:**
- Percentage of checkout API requests returning 2xx/3xx (non-server-error)
- Target: 99.95% over 30 days

**Latency SLI:**
- Percentage of checkout requests completing in < 2 seconds
- Target: 95% of requests under 2s over 30 days

**Data freshness SLI (if applicable):**
- Percentage of inventory checks returning data updated within 60 seconds
- Target: 99.9% of inventory checks are fresh

**Why not just uptime?** Uptime doesn't capture partial failures. A service can be '100% available' but serving 8-second responses — users are effectively experiencing downtime. SLIs based on latency and correctness capture the real user experience.

---

### Q9. A service's error budget is at 5% remaining with 2 weeks left in the month. What do you do?

**[YOUR ANSWER]:**
This is a classic SRE decision point. With 5% error budget remaining and 14 days left, we're at high risk of SLO breach.

**Immediate actions:**
1. Alert the service team and engineering leadership
2. Implement an error budget policy — freeze all non-critical feature deployments for this service
3. Audit recent deployments — was there a specific change that consumed budget?
4. Review the last 2 weeks of incidents for this service
5. Check if there are known reliability improvements pending that could be prioritized

**Medium-term actions:**
1. Prioritize reliability work over feature work for this service
2. Implement better canary deployments to reduce deployment risk
3. Add more defensive coding — circuit breakers, timeouts, retries with exponential backoff
4. Review the SLO target — is 99.9% the right target for this service's criticality?

**At Vendasta:** We have a formal error budget policy. When budget drops below 50%, deployment freeze is recommended. Below 20%, deployment freeze is mandatory for that service. This creates accountability — developers own the reliability of their services because unreliable code directly blocks their future feature deployments.

---

*Next: Move to File 08 — System Design & Final Round*
