# End-to-End Workflow — Complete SRE/DevOps Walkthrough (6 YOE)

---

## The Complete Picture

This document traces a **single feature** from developer's laptop to production with full observability. This is the story you tell in interviews when asked about your workflow.

---

## Full E2E Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: DEVELOPMENT                                                        │
│                                                                              │
│  Developer's Machine                                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  1. git checkout -b feature/add-payment-retry                         │  │
│  │  2. Write code + unit tests                                            │  │
│  │  3. docker-compose up (local testing with real DB, Redis)              │  │
│  │  4. pre-commit hooks run: go fmt, go vet, detect-secrets               │  │
│  │  5. git push origin feature/add-payment-retry                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                │
                │ git push → GitHub webhook → Jenkins trigger
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: CI (Continuous Integration)  — Jenkins / GitHub Actions            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  On PR Open/Push:                                                     │   │
│  │                                                                       │   │
│  │  [Checkout] → [go test ./...] → [golangci-lint] → [docker build]     │   │
│  │       │              │                │                  │            │   │
│  │       │          FAIL: PR             │           [trivy scan]        │   │
│  │       │          blocked              │                  │            │   │
│  │       │                         FAIL: PR          [docker push        │   │
│  │       │                         blocked            gcr.io/proj/       │   │
│  │       │                                            app:$GIT_SHA]      │   │
│  │       └─────────────── All pass ──────────────────────►              │   │
│  │                         GitHub Status Check = GREEN                   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                │
                │ PR Review + Approval + Status Green → Merge to main
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: CD — STAGING DEPLOYMENT                                            │
│                                                                              │
│  Merge to main → Jenkins (main branch pipeline):                             │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  1. docker push gcr.io/proj/app:$GIT_SHA  (image already built)      │   │
│  │  2. Update Helm values / K8s manifest with new image tag              │   │
│  │  3. kubectl apply -n staging                                          │   │
│  │  4. kubectl rollout status deployment/myapp -n staging --timeout=5m   │   │
│  │  5. Run integration test suite against staging URL                    │   │
│  │     ├─ API contract tests                                             │   │
│  │     ├─ Smoke tests (critical paths)                                   │   │
│  │     └─ Performance test (ensure P99 < 500ms)                          │   │
│  │  6. Slack: "✅ myapp v1.2.3 deployed to staging, tests passed"        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                │
                │ Manual approval gate (Jenkins input / GitHub environment)
                │ Or: automated if all gates pass (continuous delivery)
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: PRODUCTION DEPLOYMENT — Zero Downtime Rolling Update               │
│                                                                              │
│  GKE Cluster (wsp-prod)                                                      │
│                                                                              │
│  BEFORE:                                                                     │
│  ReplicaSet-OLD (3 pods)     [pod-1 v1.1] [pod-2 v1.1] [pod-3 v1.1]       │
│  Service → all 3 pods (load balanced)                                        │
│                                                                              │
│  DURING (Rolling Update, maxUnavailable=0, maxSurge=1):                      │
│  ReplicaSet-OLD (2 pods)     [pod-1 v1.1] [pod-2 v1.1]                     │
│  ReplicaSet-NEW (2 pods)     [pod-4 v1.2 STARTING] [pod-5 v1.2 READY]      │
│  Service → pod-1, pod-2, pod-5 (readiness probe passed)                     │
│                                                                              │
│  AFTER:                                                                      │
│  ReplicaSet-OLD (0 pods)     (scaled down, still exists for rollback)        │
│  ReplicaSet-NEW (3 pods)     [pod-4 v1.2] [pod-5 v1.2] [pod-6 v1.2]       │
│  Service → all 3 new pods                                                    │
│                                                                              │
│  Timeline: ~3-5 minutes for 3-replica rolling update                         │
│                                                                              │
│  Post-deploy checks (Jenkins):                                               │
│  - kubectl rollout status (exit 0 = success)                                 │
│  - Datadog synthetic check: critical API endpoint returns 200                │
│  - Error rate check: below threshold for 5 min after deploy                  │
└─────────────────────────────────────────────────────────────────────────────┘
                │
                │ Continuous
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: OBSERVABILITY — Normal Operations                                  │
│                                                                              │
│  Prometheus scrapes every 30s:                                               │
│    /metrics from all pods → TSDB storage → evaluate alert rules              │
│                                                                              │
│  Datadog Agent (DaemonSet):                                                  │
│    Metrics → DD Cloud (infra, K8s state)                                     │
│    Traces  → DD APM (request flows, latency breakdown)                       │
│    Logs    → DD Log Mgmt (structured, searchable, correlated)                │
│                                                                              │
│  Grafana Dashboards (auto-refresh every 30s):                                │
│    ├─ RED metrics: RPS, Error Rate, P99 Latency                              │
│    ├─ USE metrics: CPU%, Memory%, Pod Restarts                               │
│    └─ SLO panel: Error budget burn rate                                      │
│                                                                              │
│  Cloud Logging:                                                               │
│    All K8s container logs → Cloud Logging → Archived to GCS after 30 days   │
└─────────────────────────────────────────────────────────────────────────────┘
                │
                │ Alert fires!
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE 6: INCIDENT RESPONSE                                                  │
│                                                                              │
│  02:34 AM — Alertmanager fires: HighErrorRate (>1%) for myapp/production    │
│                 │                                                             │
│                 ▼                                                             │
│  02:34 AM — PagerDuty pages on-call SRE (Chethan)                           │
│                 │                                                             │
│                 ▼                                                             │
│  02:36 AM — SRE acknowledges, opens Datadog dashboard                        │
│             Sees: error spike started at 02:33, correlates to deploy at 02:30│
│                 │                                                             │
│                 ▼                                                             │
│  02:37 AM — Check APM trace: NullPointerException in PaymentRetryService     │
│             Check logs: "failed to deserialize legacy payment format"         │
│                 │                                                             │
│                 ▼                                                             │
│  02:38 AM — ROLLBACK:                                                         │
│             kubectl rollout undo deployment/myapp -n production               │
│             kubectl rollout status deployment/myapp -n production             │
│                 │                                                             │
│                 ▼                                                             │
│  02:41 AM — Error rate back to 0%. Alertmanager resolves.                    │
│             PagerDuty incident resolved. Slack notification sent.             │
│                 │                                                             │
│                 ▼                                                             │
│  Same day  — Root cause analysis:                                             │
│             Bug in deserialization for payments with old format.              │
│             Fix applied, unit test added, re-deployed after review.           │
│                 │                                                             │
│                 ▼                                                             │
│  Next week — Postmortem published:                                            │
│             Timeline, root cause, impact, action items:                       │
│             1. Add payment format test in CI                                  │
│             2. Add canary deployment (test 5% traffic before full rollout)    │
│             3. Improve pre-deploy smoke test to catch serialization errors    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## How All Tools Interconnect

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      TOOL INTEGRATION MAP                                   │
│                                                                             │
│  GitHub ──────────────────────────────────────────────────────────────┐    │
│    │ Webhooks trigger CI                                               │    │
│    │ CODEOWNERS enforce review                                         │    │
│    │ Branch protection enforce quality gates                           │    │
│    ▼                                                                   │    │
│  Jenkins ─────────────────────────────────────────────────────────┐   │    │
│    │ Pulls code from GitHub                                        │   │    │
│    │ Runs tests in K8s pod agents (GKE)                           │   │    │
│    │ Builds Docker images                                          │   │    │
│    │ Pushes to Artifact Registry (GCP)                            │   │    │
│    │ Deploys to GKE via kubectl                                   │   │    │
│    │ Notifies Slack on success/failure                            │   │    │
│    │ Sends deploy events to Datadog                               │   │    │
│    ▼                                                              │   │    │
│  Docker ──────────────────────────────────────────────────────┐  │   │    │
│    │ Packages app into immutable artifact                      │  │   │    │
│    │ Pushed to GCR/Artifact Registry                          │  │   │    │
│    │ GKE pulls image on deploy                                │  │   │    │
│    ▼                                                          │  │   │    │
│  GKE ─────────────────────────────────────────────────────┐  │  │   │    │
│    │ Runs containers (Docker images)                       │  │  │   │    │
│    │ Exposes via GCP LoadBalancer                         │  │  │   │    │
│    │ Integrates with GCP IAM (Workload Identity)          │  │  │   │    │
│    │ Writes logs to Cloud Logging                         │  │  │   │    │
│    │ Exposes /metrics → Prometheus scrapes               │  │  │   │    │
│    │ Runs Datadog Agent DaemonSet                        │  │  │   │    │
│    ▼                                                     │  │  │   │    │
│  Prometheus ──────────────────────────────────────────┐  │  │  │   │    │
│    │ Scrapes metrics from GKE pods                    │  │  │  │   │    │
│    │ Evaluates alert rules                            │  │  │  │   │    │
│    │ Fires to Alertmanager                           │  │  │  │   │    │
│    │ Queried by Grafana for dashboards               │  │  │  │   │    │
│    ▼                                                │  │  │  │   │    │
│  Grafana ────────────────────────────────────────┐  │  │  │  │   │    │
│    │ Queries Prometheus                          │  │  │  │  │   │    │
│    │ Displays dashboards                        │  │  │  │  │   │    │
│    │ Sends alerts to PagerDuty/Slack           │  │  │  │  │   │    │
│    └────────────────────────────────────────── ┘  │  │  │  │   │    │
│  Datadog ──────────────────────────────────────┐   │  │  │  │   │    │
│    │ Collects metrics, traces, logs from GKE  │   │  │  │  │   │    │
│    │ Provides APM (distributed traces)        │   │  │  │  │   │    │
│    │ Monitors (alerts) → PagerDuty            │   │  │  │  │   │    │
│    │ SLO tracking                             │   │  │  │  │   │    │
│    └──────────────────────────────────────────┘   │  │  │  │   │    │
└───────────────────────────────────────────────────┘  └──┘  └───┘    │
                                                                        │
GCP Services used throughout:                                           │
  GCS → store Docker layers cache, Terraform state, backups             │
  Cloud SQL → application database (accessed via proxy in GKE)          │
  Secret Manager → DB passwords, API keys (no plaintext secrets)        │
  Pub/Sub → async messaging between services                            │
  Cloud Armor → WAF/DDoS protection on the LB                          │
  Cloud Logging → unified log sink (K8s + app logs)                     │
  IAM → Workload Identity for pod → GCP API auth                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Typical Day in the Life (SRE)

### Morning
```
09:00 - Check overnight alerts (Datadog/PagerDuty)
09:15 - Review SLO dashboards: are we burning error budget?
09:30 - Team standup: any incidents last night? toil items?
09:45 - Review pending PRs (infrastructure, runbooks, config changes)
```

### Active Work
```
10:00 - Work on reliability improvements (reduce toil, automate)
      - Might be: writing a new alert rule, tuning HPA, adding chaos test
11:00 - Code review: Jenkins pipeline change, K8s manifest update
13:00 - Deploy new service version to staging, observe metrics
14:00 - Incident review: postmortem writing from yesterday's incident
15:00 - Capacity planning: check next 30-day growth forecast, node pool sizing
16:00 - On-call handoff: brief incoming on-call about active issues
```

### On-Call Rotation
```
Pagerduty rotation: 1 week primary, 1 week secondary
Response time SLA: Critical=5min, Warning=30min
Runbook for every alert (or the alert shouldn't exist)
Post-incident: always write blameless postmortem
```

---

## Deployment Strategies Comparison

```
┌──────────────────────────────────────────────────────────────────────────┐
│  STRATEGY          │ RISK │ ROLLBACK │ TRAFFIC SPLIT │ USE CASE          │
│──────────────────────────────────────────────────────────────────────────│
│  Recreate          │ HIGH │ Manual   │ None (downtime)│ Dev/test only     │
│  Rolling Update    │ MED  │ Fast     │ Gradual        │ Most prod cases   │
│  Blue/Green        │ LOW  │ Instant  │ Full switch    │ Risky changes     │
│  Canary            │ LOW  │ Fast     │ %, gradual     │ High-traffic prod │
│  A/B Testing       │ LOW  │ Fast     │ By user segment│ Feature testing   │
└──────────────────────────────────────────────────────────────────────────┘

Canary with Istio/Flagger:
  Step 1:  5% traffic to v1.2
  Step 2: 20% (if error rate ok)
  Step 3: 50%
  Step 4: 100% (promote)
  Any step: error rate > 1% → auto rollback to v1.1
```

---

## Infrastructure as Code Flow

```
┌────────────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE AS CODE                                         │
│                                                                │
│  Terraform (GCP infrastructure):                               │
│  ├─ VPC, subnets, firewall rules                               │
│  ├─ GKE cluster + node pools                                   │
│  ├─ Cloud SQL instances                                        │
│  ├─ GCS buckets                                                │
│  ├─ IAM bindings                                               │
│  └─ Secret Manager secrets (keys only, not values)             │
│                                                                │
│  Helm Charts (K8s application config):                         │
│  ├─ Deployment, Service, Ingress                               │
│  ├─ HPA, PDB                                                   │
│  ├─ ConfigMaps                                                 │
│  └─ ServiceMonitor (Prometheus)                                │
│                                                                │
│  GitOps with ArgoCD (alternative to Jenkins deploy):           │
│  ├─ Git repo = source of truth for K8s state                  │
│  ├─ ArgoCD watches Git, auto-syncs to cluster                 │
│  ├─ Drift detection: alerts if cluster != git state           │
│  └─ PR-based deployments (merge = deploy)                     │
│                                                                │
│  Workflow:                                                     │
│  Dev changes Helm values → PR → Review → Merge → ArgoCD       │
│  auto-deploys within seconds                                   │
└────────────────────────────────────────────────────────────────┘
```

---

## The 12-Factor App Principles (Context for Deployments)

| Factor | Practice |
|--------|---------|
| Codebase | One repo per service, tracked in git |
| Dependencies | Explicit in go.mod / package.json / requirements.txt |
| Config | From environment (K8s ConfigMaps/Secrets), never in code |
| Backing Services | Treat DB, Redis, queues as attached resources (config URL) |
| Build/Release/Run | Separate: CI builds → image tagged → deploy to env |
| Processes | Stateless. State in backing services (GCS, CloudSQL) |
| Port Binding | Self-contained HTTP server, expose via port |
| Concurrency | Scale out with more pods (HPA) |
| Disposability | Fast startup, graceful shutdown (SIGTERM handling) |
| Dev/Prod Parity | Same Docker image from dev → staging → prod |
| Logs | Treat as event streams to stdout (collected by Datadog/CL) |
| Admin Processes | Run as K8s Jobs, not ad-hoc SSH sessions |

---

## Common Interview Scenario: "Walk me through a deployment"

**Answer framework (use this in interviews):**

1. **Code**: Developer pushes to GitHub, opens PR
2. **CI**: Jenkins/GH Actions runs tests, builds Docker image, scans for CVEs, pushes to Artifact Registry
3. **Review**: At least 1 CODEOWNER approves, status checks green
4. **Merge**: Triggers CD pipeline
5. **Staging**: Auto-deploy to staging, integration tests run
6. **Gate**: Manual approval (or automated for low-risk changes)
7. **Production**: Rolling update in GKE (`kubectl set image` or `helm upgrade`)
8. **Verify**: Monitor Grafana/Datadog for 15 min — error rate, latency, restarts
9. **Alert**: If error rate spikes, auto-rollback (`kubectl rollout undo`)
10. **Success**: Slack notification, Jira ticket closed

> Key points to emphasize: zero-downtime rolling update, readiness probes, PDB, automatic rollback, observability from minute one.
