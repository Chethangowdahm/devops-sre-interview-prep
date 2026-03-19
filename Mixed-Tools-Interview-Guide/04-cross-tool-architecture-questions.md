# Mixed Tools Interview Guide - Part 4: Cross-Tool Architecture Questions

> These questions require deep knowledge of multiple tools working together.
> Senior SRE/DevOps roles always ask architecture-level questions. Show you can connect the dots.

---

## ARCHITECTURE QUESTION 1

**[INTERVIEWER]:** "Design the complete observability stack for a microservices platform with 20+ services on Kubernetes. What tools would you use and why?"

**[TOOLS INVOLVED: Prometheus + Grafana + Datadog + ELK/Loki + Jaeger/Datadog APM + Kubernetes]**

**YOUR ANSWER:**
"I design observability in three pillars - metrics, logs, and traces - plus a fourth pillar: alerting and incident response.

**PILLAR 1: METRICS**

For infrastructure metrics:
- Prometheus Operator on the cluster for K8s-native metrics collection
- node-exporter DaemonSet for host-level metrics (CPU, memory, disk, network per node)
- kube-state-metrics for K8s object health (pod states, deployment rollout status, PVC binding)
- Cluster Autoscaler metrics for scaling decisions

For application metrics:
- Every service exposes /metrics in Prometheus format
- ServiceMonitor CRDs define what Prometheus scrapes
- Custom metrics for business KPIs (orders processed per second, revenue per minute)

Visualization:
- Grafana with standardized dashboards - one RED dashboard template per service, one USE template per node
- Grafana on-call for dashboard-driven alerting routing

**PILLAR 2: LOGS**

Structured JSON logging everywhere:
```json
{"level":"error","service":"payment-api","trace_id":"abc123","user_id":"u456","message":"payment declined","error":"insufficient funds","timestamp":"2024-01-15T03:45:00Z"}
```

Log collection: Fluent Bit DaemonSet (lightweight) collects from /var/log/containers and ships to:
- Loki (self-hosted, cost-effective for high volume) OR
- Datadog Log Management (better search, more expensive)

Log retention: 30 days hot (Loki/Datadog), 1 year cold (GCS/S3 archive)

**PILLAR 3: TRACES**

Distributed tracing with Datadog APM:
- Every service instruments with dd-trace library
- Trace context propagated via HTTP headers (traceparent/tracestate - W3C standard)
- All traces correlated via trace_id
- Service map auto-generated from trace data

Flame graphs show: exactly where time is spent in each request across all service hops.

Trace sampling: 100% of error traces (never drop errors), 10% of success traces (cost control)

**PILLAR 4: ALERTING AND INCIDENT RESPONSE**

Alert hierarchy:
- P1 (service down, SLO breach): PagerDuty + Slack + phone call
- P2 (high error rate, latency spike): PagerDuty + Slack
- P3 (warning threshold): Slack only, no page

Alert quality rules:
- Every alert links to a runbook in Confluence
- Alerts are SLO-based (burn rate), not threshold-based (CPU > 80%)
- Alert review quarterly: remove any alert that fired and required no action

Incident management: Datadog Incident Management for P1/P2 - automatic timeline, stakeholder communication, action item tracking.

**At Vendasta:** This exact stack. The key insight: Datadog for application observability (APM, logs, SLOs) and Prometheus+Grafana for infrastructure. Two tools for different purposes, not one-size-fits-all."

---

## ARCHITECTURE QUESTION 2

**[INTERVIEWER]:** "How would you implement a blue/green deployment strategy on GKE using ArgoCD and ensure zero downtime with database schema changes?"

**[TOOLS INVOLVED: ArgoCD + Kubernetes + GCP + Databases]**

**YOUR ANSWER:**
"Blue/green on Kubernetes requires careful orchestration especially with DB changes. Here is the full approach:

**Kubernetes blue/green setup:**
```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api-blue
  labels:
    version: blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: payment-api
      version: blue
  template:
    metadata:
      labels:
        app: payment-api
        version: blue
    spec:
      containers:
      - name: payment-api
        image: gcr.io/vendasta-prod/payment-api:v1.0.0
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api-green
  labels:
    version: green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: payment-api
      version: green
  template:
    metadata:
      labels:
        app: payment-api
        version: green
    spec:
      containers:
      - name: payment-api
        image: gcr.io/vendasta-prod/payment-api:v2.0.0
---
# Service - switch traffic by changing selector
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  selector:
    app: payment-api
    version: blue    # Change to 'green' to switch all traffic
  ports:
  - port: 80
    targetPort: 8080
```

**ArgoCD management:**
1. Deploy green version without switching service selector - zero traffic to green
2. Verify green pods are all Running and passing health checks
3. Run smoke tests against green directly (using port-forward or internal URL)
4. Update service selector from blue to green in Git - ArgoCD applies it
5. Monitor for 15 minutes - all traffic now on green
6. If issues: switch selector back to blue in Git - instant rollback
7. After confidence: scale down blue deployment to 0 (keep it for 24 hours as safety net)

**Handling database schema changes (the hard part):**

The expand-contract (parallel change) pattern:

Phase 1 - Expand: Add new column/table without removing old ones. Both blue (old code) and green (new code) can run simultaneously:
```sql
-- Migration: Add new column with default (non-breaking)
ALTER TABLE orders ADD COLUMN customer_notes TEXT DEFAULT '';
-- Blue code ignores this column, green code writes to it
```

Phase 2 - Deploy green: New code writes to BOTH old and new columns during transition

Phase 3 - Verify: All traffic on green, old column no longer read

Phase 4 - Contract: Remove old column in a LATER release (minimum 1 full deployment cycle later):
```sql
-- Safe to drop ONLY after confirming no old pods read this column
ALTER TABLE orders DROP COLUMN old_customer_field;
```

**What NOT to do in blue/green:**
- NEVER: DROP COLUMN before switching to green (breaks blue pods still running)
- NEVER: RENAME COLUMN (breaks blue pods)
- NEVER: NOT NULL without DEFAULT (blocks both versions from writing)

**At Vendasta:** We use ArgoCD Rollouts with blue/green strategy for our database-heavy services. The key learning: DB migrations are deployed BEFORE the new application code, not simultaneously. Our CI gate verifies migrations are backward-compatible before allowing deploy."

---

## ARCHITECTURE QUESTION 3

**[INTERVIEWER]:** "How would you implement cost optimization for a GKE + GCP workload that is spending $50,000/month on infrastructure?"

**[TOOLS INVOLVED: GCP + Kubernetes + Terraform + Prometheus/Monitoring]**

**YOUR ANSWER:**
"$50K/month is significant. Cost optimization has multiple levers. Here is my systematic approach:

**Step 1: Understand where the money is going**
```bash
# GCP Billing breakdown by resource
gcloud billing accounts list
# In GCP Console: Billing -> Cost Breakdown -> by SKU, by project, by label

# Kubernetes resource efficiency
kubectl top pods -A --sort-by=cpu
kubectl top nodes

# VPA recommendations (shows actual vs requested vs limit)
kubectl get vpa -A -o yaml | grep recommendation -A 10
```

**Optimization Lever 1: Right-size pod resource requests (biggest win, often 30-40% savings)**

Most teams over-request resources out of caution. VPA in recommendation mode shows actual usage:
```bash
# Deploy VPA in recommendation mode (no auto-changes, just recommendations)
kubectl apply -f https://github.com/kubernetes/autoscaler/raw/master/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml
```

If a pod requests 1 CPU but only uses 100m, you are wasting 900m per pod. For 100 pods, that is 90 vCPUs wasted.

**Optimization Lever 2: Use Spot/Preemptible nodes for non-critical workloads (60-80% cheaper)**
```hcl
# Terraform - add spot node pool
resource "google_container_node_pool" "spot_workers" {
  name    = "spot-workers"
  cluster = google_container_cluster.main.name

  node_config {
    machine_type = "n2-standard-4"
    spot         = true  # 60-80% cheaper, may be preempted with 30s notice
  }

  autoscaling {
    min_node_count = 0
    max_node_count = 50
  }
}
```

Label stateless services (web APIs, batch jobs) to prefer spot nodes via node affinity. Keep stateful workloads (databases, caches) on regular nodes.

**Optimization Lever 3: GKE Cluster Autoscaler + scale-to-zero**
- Enable cluster autoscaler so idle node pools scale down to 0 during low traffic
- For dev/staging: schedule scale-down overnight (nobody needs dev at 3 AM)
```bash
# Cron job to scale dev to 0 at night
kubectl create cronjob scale-down-dev \
  --image=bitnami/kubectl \
  --schedule='0 22 * * 1-5' \
  --restart=Never \
  -- kubectl scale deployment --all --replicas=0 -n development
```

**Optimization Lever 4: Storage and networking costs**
- Delete unused PVCs (orphaned volumes still charge)
- Use regional storage class only when needed (2x cheaper than multi-regional for non-critical data)
- Use Cloud NAT efficiently - egress charges add up
- Enable Cloud Storage lifecycle policies to move old data to cheaper tiers (Nearline -> Coldline -> Archive)

**Optimization Lever 5: Committed Use Discounts (CUDs) for predictable workloads**
- GCP gives 37-55% discount for 1 or 3 year commitments on compute
- Use Terraform to manage CUD purchases
- Only commit what you are confident you will use (base load, not peak)

**Expected savings breakdown for a $50K/month bill:**
- Right-sizing resources: -$8,000 (16% reduction)
- Spot nodes for 50% of workload: -$12,000 (24% reduction)
- Dev/staging overnight scale-down: -$3,000 (6% reduction)
- Storage optimization: -$2,000 (4% reduction)
- CUDs for base load: -$7,000 (14% reduction)
- Total potential savings: ~$32,000/month (64% reduction)

**At Vendasta:** I led a cost optimization initiative that reduced our GCP bill by 40% over 3 months. The biggest win was VPA recommendations - our developers had requested 10x more memory than services actually used. Combined with spot nodes for our batch processing workloads, we saved approximately $15,000/month."

---

## ARCHITECTURE QUESTION 4

**[INTERVIEWER]:** "How do you handle configuration management across multiple environments - dev, staging, production - in a Kubernetes/GitOps setup?"

**[TOOLS INVOLVED: Kubernetes + Helm + ArgoCD + Secret Management + Git]**

**YOUR ANSWER:**
"Config management in GitOps needs to be rigorous because config errors cause production incidents as often as code bugs.

**My configuration hierarchy:**

Level 1 - Base Helm chart (common across all environments):
```yaml
# helm/payment-api/values.yaml (base)
replicaCount: 1
image:
  repository: gcr.io/vendasta-prod/payment-api
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 100m
    memory: 128Mi

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
```

Level 2 - Environment-specific values override the base:
```yaml
# helm/payment-api/values-production.yaml
replicaCount: 5
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi

hpa:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 70
```

```yaml
# helm/payment-api/values-staging.yaml
replicaCount: 2
resources:
  requests:
    cpu: 200m
    memory: 256Mi

hpa:
  enabled: false  # No autoscaling in staging
```

Level 3 - Secrets (NEVER in Git):
- Stored in GCP Secret Manager
- Synced to K8s Secrets via External Secrets Operator
- ArgoCD ignores secret values in diff (they change frequently)

**ArgoCD Application per environment:**
```yaml
# ArgoCD Application for production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api-production
spec:
  source:
    repoURL: https://github.com/vendasta/k8s-manifests
    path: helm/payment-api
    helm:
      valueFiles:
      - values.yaml                # Base
      - values-production.yaml     # Production overrides
  destination:
    namespace: production
```

**Feature flags for gradual rollouts:**
For risky configuration changes, use feature flags instead of direct config changes:
- LaunchDarkly or GCP Feature Flags
- New behavior enabled for 5% of users first, then 25%, then 100%
- Roll back = disable the flag (no deployment needed)

**Config change review process:**
- All config changes go through PR with the same review process as code
- Config-only changes require 1 reviewer (lower risk than code changes)
- Changes affecting production resources require 2 reviewers
- ArgoCD shows exact diff before sync - reviewers see what will change in the cluster

**At Vendasta:** I standardized our config structure across all 15 microservices. Before my changes, every service had different patterns - some used ConfigMaps directly, some had env vars hardcoded in the deployment. The unified Helm-based approach reduced config-related incidents by 80% in the first 3 months."

---

## ARCHITECTURE QUESTION 5

**[INTERVIEWER]:** "What is your approach to on-call rotation and incident management? How do you ensure the team doesn't burn out?"

**[TOOLS INVOLVED: Datadog + PagerDuty + SRE practices]**

**YOUR ANSWER:**
"On-call sustainability is critical for team health. Here is my full approach:

**Alert quality first - most burnout comes from noisy alerts**

Monthly alert audit process:
1. Pull all alerts that fired in the last 30 days from PagerDuty/Datadog
2. Categorize each: Actionable (required immediate engineer action) vs Noise (auto-resolved, no action needed)
3. Any alert that was noise more than 20% of the time gets reviewed and fixed or deleted
4. Target: on-call engineer gets fewer than 5 pages per shift that require real action

**Runbooks for everything that pages**

Every alert must have a runbook:
```markdown
# Runbook: PaymentAPI High Error Rate

## Summary
This alert fires when error rate exceeds 1% for 5 minutes.

## Impact
- Users cannot complete purchases
- Revenue impact: approximately $500/minute

## Diagnosis
1. Check ArgoCD - was there a recent deployment?
2. Check Datadog APM traces - which endpoint is failing?
3. Check downstream services: database, Stripe, notification service

## Fix options
- Recent deployment: rollback via ArgoCD (2 minutes)
- Database issue: see database runbook
- Stripe API down: activate maintenance mode

## Escalation
- If not resolved in 15 minutes: page engineering manager
- If revenue impact >$10k: page VP Engineering
```

**On-call rotation structure:**
- Minimum 5 people in rotation (weekly shifts)
- Shadow on-call for new engineers (learn before owning)
- Explicit handoff: outgoing on-call writes a shift summary and unresolved action items
- Post-incident: on-call engineer gets recovery time (if woken up at 3 AM, start later the next day)

**Error budget policy prevents long-term burnout:**
- If a service's error budget is exhausted, feature work stops
- Engineers who write the service fix its reliability - no separate firefighting team
- This creates the right incentive: reliable code means peaceful on-call

**Blameless postmortems:**
- Every P1/P2 gets a written postmortem within 48 hours
- Shared with entire engineering org - everyone learns
- Action items are tracked, assigned, and have due dates
- No blame, no firing - incidents are system failures, not individual failures

**At Vendasta:** When I joined, on-call had 15+ pages per shift. After 3 months of alert auditing and runbook creation, it dropped to under 5. On-call became manageable instead of dreaded. Engineer satisfaction scores for on-call improved significantly."

---

## QUICK-FIRE MIXED QUESTIONS (Speed Round)

---

**Q: What is the difference between a K8s Service of type ClusterIP, NodePort, and LoadBalancer?**

ClusterIP: Internal only, accessible within cluster via DNS. NodePort: Exposes service on each node's IP at a static port (30000-32767). LoadBalancer: Provisions a cloud load balancer (GCP LB, AWS ELB) with external IP. In production, use LoadBalancer or Ingress - never expose NodePort directly.

---

**Q: You run `kubectl exec -it pod -- bash` and try to run `curl` but it's not installed. What do you do?**

Use `kubectl debug` to attach an ephemeral debug container with tools:
```bash
kubectl debug -it <pod-name> --image=curlimages/curl --target=<container-name>
```
Or use `kubectl run` to create a temporary debug pod:
```bash
kubectl run debug --image=curlimages/curl --rm -it -- sh
```
Never install debugging tools in production images - it increases attack surface.

---

**Q: What is the difference between Ansible and Terraform? When do you use each?**

Terraform: declarative, idempotent, manages INFRASTRUCTURE STATE (cloud resources: VMs, networks, databases). It knows what exists and reconciles to desired state.

Ansible: procedural (though idempotent), manages CONFIGURATION AND SOFTWARE on existing machines (install packages, configure services, copy files). It runs tasks in order.

Rule of thumb: Terraform to provision the server, Ansible to configure it. In cloud-native K8s environments, Ansible is less needed - you bake everything into Docker images instead.

---

**Q: How do you debug a Kubernetes network policy that is blocking traffic?**

```bash
# Check if network policies exist in the namespace
kubectl get networkpolicy -n <namespace>

# Describe to see the rules
kubectl describe networkpolicy <name> -n <namespace>

# Test connectivity from inside a pod
kubectl exec -it <source-pod> -- curl <target-service>:<port>

# Temporarily delete network policy to confirm it is the culprit
kubectl delete networkpolicy <name> -n <namespace>
# If traffic flows -> network policy was blocking it
# Re-apply with fixed rules
```
With Cilium CNI: `cilium connectivity test` provides comprehensive network connectivity diagnostics.

---

**Q: What is the 12-Factor App methodology and how does it relate to container/K8s best practices?**

The 12 factors most relevant to containers/K8s:
- Factor 3 (Config): Store config in environment variables, not code - maps to K8s ConfigMaps and Secrets
- Factor 6 (Processes): Stateless processes - maps to K8s Deployments (pods can be killed and recreated)
- Factor 7 (Port binding): Service is self-contained, exports via port - maps to K8s Service
- Factor 8 (Concurrency): Scale via process model - maps to K8s HPA
- Factor 9 (Disposability): Fast startup, graceful shutdown - maps to liveness probes and terminationGracePeriod
- Factor 11 (Logs): Write to stdout/stderr - Kubernetes captures this automatically for log aggregation

---

**Q: You need to migrate a stateful service (PostgreSQL) from one GKE cluster to another with minimal downtime. How?**

Strategy: logical replication for near-zero downtime
1. Set up logical replication from old DB to new DB (new DB acts as subscriber)
2. Wait for replication lag to reach near-zero
3. Maintenance window: stop writes to old DB (put app in read-only mode or maintenance page)
4. Wait for replication to fully catch up (seconds)
5. Promote new DB as primary
6. Update application connection string to point to new DB (via K8s Secret update + rolling restart)
7. Verify: test writes on new DB
8. Keep old DB running for 24 hours as fallback

For Cloud SQL on GCP: use Cloud SQL Migration Service which handles most of this automatically.

---

## FINAL MASTER TIPS

**How to answer ANY unknown question:**

Use the STAR + Technical framework:
1. Start with the PRINCIPLE (what concept applies here?)
2. Apply to a SPECIFIC TOOL you know (how does this tool implement it?)
3. Give a REAL EXAMPLE from your work (what did YOU actually do?)
4. Mention the TRADEOFF or ALTERNATIVE (shows senior thinking)

Example: "What is X?"
"X is a [principle/pattern]. In [tool I use], it works by [specific mechanism]. At Vendasta, I [specific thing I did]. The tradeoff compared to alternative Y is [specific comparison]."

**Phrases that impress:**
- 'In production, what I found matters most is...'
- 'The failure mode here is... which is why I always...'
- 'We had an incident where... and the lesson was...'
- 'The tradeoff between X and Y is... we chose X because...'
- 'At Vendasta's scale of 60,000 businesses, this matters because...'

**You have all the answers. Trust your 6.2 years. Go get that offer.**
