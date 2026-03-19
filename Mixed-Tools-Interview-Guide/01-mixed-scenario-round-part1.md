# Mixed Tools Interview Guide - Part 1: Real Interview Flow (Mixed Q&A)

> How to use: This simulates exactly how a senior interviewer mixes topics in a real interview.
> They jump between tools to test depth. Read each Q out loud, cover the answer, recall it, then check.

---

## ROUND 1: WARM-UP (First 10 minutes - Breadth Check)

---

### Q1. [INTERVIEWER]: "You mentioned you work with GKE and ArgoCD at Vendasta. Walk me through what happens from the moment a developer pushes code to GitHub until it runs in production."

**[TOOLS TESTED: Git + GitHub Actions + Docker + ArgoCD + Kubernetes]**

**YOUR ANSWER:**
"Our pipeline is fully GitOps-based. Here is the end-to-end flow:

1. Developer pushes code to a feature branch and opens a PR on GitHub.
2. GitHub Actions triggers: runs unit tests with `go test ./...`, linting, `go vet`. PR is blocked if anything fails.
3. On merge to main: second workflow builds a multi-stage Docker image. Go binary built in stage 1, copied to distroless runtime in stage 2 - image size drops from 350MB to 12MB.
4. Image pushed to Google Artifact Registry tagged with the commit SHA.
5. A third job updates the image tag in our `k8s-manifests` repo - one line change in `values-prod.yaml`.
6. ArgoCD detects the manifest change. Compares desired state (Git) vs live state (GKE cluster).
7. Staging: ArgoCD auto-syncs. Production: Slack notification in #deployments, senior engineer reviews diff and clicks Sync in ArgoCD UI.
8. ArgoCD applies the Helm chart - Kubernetes does rolling update: new pods created, pass readiness probes, old pods terminated.
9. Datadog monitors error rate and latency for 15 min post-deploy. If anything spikes, one-click rollback in ArgoCD.

Total time: merge to staging ~4 minutes, merge to production ~10-15 minutes including review."

---

### Q2. [INTERVIEWER]: "Good. Your monitoring shows a memory spike right after that deployment. What is your next move?"

**[TOOLS TESTED: Datadog + Kubernetes + Docker + ArgoCD]**

**YOUR ANSWER:**
"First, correlate timing: did the spike start when new pods came up? If yes, almost certainly the new code.

Step 1 - Assess severity: Are pods OOMKilling?
```bash
kubectl top pods -n production --sort-by=memory
kubectl describe pod <pod-name> -n production | grep -A5 'Last State'
# Look for: Reason: OOMKilled
```

Step 2 - Check Datadog APM flame graph: Which function allocates memory? New cache, goroutine leak, or unbounded slice?

Step 3 - Decision tree:
- Pods OOMKilling -> immediate rollback via ArgoCD
- Memory high but stable -> temporary limit increase, investigate async
- Memory slowly growing -> likely a leak, rollback proactively

Step 4 - After rollback: open a post-mortem draft, assign engineer to profile new code with `pprof` in staging under load.

**At Vendasta:** A Node.js service introduced an event emitter leak in v2.1.0 - leaked listeners on every request. Heap grew 10MB/hour. Caught within 20 minutes via Datadog, rolled back before any OOMKill in production."

---

### Q3. [INTERVIEWER]: "How does your Terraform fit into this picture? Who manages the GKE cluster itself?"

**[TOOLS TESTED: Terraform + GCP + Kubernetes]**

**YOUR ANSWER:**
"The GKE cluster is fully managed by Terraform in a separate `infra` repository.

```hcl
module "gke_cluster" {
  source  = "./modules/gke"
  project = var.project_id
  region  = "us-central1"

  node_pools = [
    {
      name         = "general"
      machine_type = "n2-standard-4"
      min_count    = 2
      max_count    = 20
      autoscaling  = true
    },
    {
      name      = "high-memory"
      machine_type = "n2-highmem-8"
      min_count = 0
      max_count = 5
      taint     = "dedicated=highmem:NoSchedule"
    }
  ]
}
```

Changes to cluster config go through PR review. GitHub Actions runs `terraform plan` and posts diff as PR comment. A senior engineer reviews before merge. Merge triggers `terraform apply` via protected workflow.

Clean separation: Terraform owns infrastructure, ArgoCD owns workloads. Both are version-controlled and auditable."

---

### Q4. [INTERVIEWER]: "Your Terraform apply accidentally deletes a node pool. What happens and how do you recover?"

**[TOOLS TESTED: Terraform + Kubernetes + GCP]**

**YOUR ANSWER:**
"What happens immediately:
- GKE drains the node pool - sends SIGTERM to all pods on those nodes
- Kubernetes tries to reschedule pods onto remaining nodes
- If remaining nodes lack capacity -> pods go Pending
- Services become degraded depending on how many replicas were lost

Recovery steps:
1. Restore the node pool immediately:
```bash
terraform apply -target=module.gke_cluster.google_container_node_pool.general
```
2. Once nodes are back, check pod scheduling:
```bash
kubectl get pods -A | grep -E 'Pending|CrashLoop'
kubectl describe pod <pending-pod> | tail -20
```
3. Check stateful workloads - PVCs need to re-attach to new nodes

Prevention I implemented:
- Add `lifecycle { prevent_destroy = true }` to node pool resources
- CI check that fails if `terraform plan` contains 'will be destroyed' for compute resources
- Datadog alert on GKE node count dropping below minimum

**At Vendasta:** A junior engineer removed a node pool selector accidentally. Plan showed 'will destroy' but they missed it. After that I added a CI gate that catches destructive compute changes and requires explicit override."

---

## ROUND 2: DEEP DIVE (Minutes 15-35)

---

### Q5. [INTERVIEWER]: "Walk me through setting up monitoring for a brand new microservice being deployed to your GKE cluster."

**[TOOLS TESTED: Kubernetes + Prometheus + Grafana + Datadog + Helm]**

**YOUR ANSWER:**
"I have a 6-layer onboarding checklist for new services:

Layer 1 - Application instrumentation (developer adds, I review in PR):
- Expose /metrics endpoint with Prometheus counters and histograms
- Expose /healthz (liveness) and /ready (readiness) endpoints

Layer 2 - Kubernetes + Prometheus integration via Helm chart values:
```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  path: /metrics

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Layer 3 - Datadog APM: add DD_SERVICE, DD_ENV, DD_VERSION env vars and DD_TRACE_AGENT_HOSTNAME pointing to hostIP (node-local Datadog agent)

Layer 4 - Define SLO before production: availability 99.9% and p99 latency <500ms configured in Datadog

Layer 5 - Grafana RED dashboard: import standard template, point at new service metrics

Layer 6 - On-call runbook: Confluence page with service description, restart procedure, common errors and fixes, escalation path

Total setup time: ~2 hours. Our Helm chart template includes all of this by default now."

---

### Q6. [INTERVIEWER]: "A developer says their service keeps getting rate-limited by an external API and it causes cascading failures. How do you solve this architecturally?"

**[TOOLS TESTED: Architecture + Kubernetes + Networking + KEDA]**

**YOUR ANSWER:**
"Classic distributed systems resilience problem. Four layers of protection:

Layer 1 - Circuit Breaker: When external API fails >50% of requests in 10s, stop sending and return cached/degraded response. After 30s, probe with 1 request - if it succeeds, gradually reopen. This prevents the cascade.

Layer 2 - Retry with Exponential Backoff and Jitter: Don't retry immediately. Wait 1s, then 2s, then 4s, with random jitter to spread retries across time and avoid thundering herd.

Layer 3 - Self-imposed Rate Limiter: Don't wait for the external API to reject us. Limit our outbound call rate proactively to 90% of their limit. Serve cached responses when limit is reached.

Layer 4 - Async Queue: For non-real-time calls, push to Pub/Sub and process with a worker pool that respects the rate limit. KEDA scales workers based on queue depth - automatically adjusts throughput.

**At Vendasta:** Notification service was hitting SendGrid rate limits during peak traffic. I implemented a Pub/Sub queue with rate-limited workers. Notifications became 100% reliable. Zero rate limit errors in production since then."

---

### Q7. [INTERVIEWER]: "Your Jenkins pipeline takes 45 minutes. How do you optimize it?"

**[TOOLS TESTED: Jenkins + Docker + Kubernetes]**

**YOUR ANSWER:**
"Profile first using Jenkins Blue Ocean stage view to see which stage is the bottleneck. Then fix in order of impact:

Fix 1 - Parallelize independent stages:
```groovy
parallel {
  stage('Unit Tests') { steps { sh 'go test ./internal/...' } }
  stage('Integration Tests') { steps { sh 'go test ./integration/...' } }
  stage('Lint') { steps { sh 'golangci-lint run' } }
}
```

Fix 2 - Fix Docker layer cache order (put stable layers first):
```dockerfile
# RIGHT: dependencies first, source last
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build
```

Fix 3 - Cache Go modules via PVC on the K8s build agent:
```groovy
volumes:
- name: go-cache
  persistentVolumeClaim:
    claimName: go-module-cache
```

Fix 4 - Add .dockerignore to reduce build context:
```
.git
node_modules
*.test
coverage/
.terraform/
```

Fix 5 - Split build and test - run tests in lightweight container, only build final Docker image after tests pass.

**At Altruist:** Reduced a 40-minute pipeline to 8 minutes: parallelizing tests (-12 min), fixing Docker cache (-10 min), caching Go modules (-7 min), .dockerignore (-3 min)."

---

### Q8. [INTERVIEWER]: "If I give you a brand new Linux server, what do you do first before putting it in production?"

**[TOOLS TESTED: Linux + Security + Ansible]**

**YOUR ANSWER:**
"I have an Ansible server-hardening playbook for this. It runs idempotently on every new server:

1. Update and patch: `apt-get upgrade -y` and enable unattended-upgrades for security patches
2. Create non-root service account, disable root SSH login
3. SSH hardening: PasswordAuthentication no, AllowUsers specific-user, MaxAuthTries 3
4. Firewall: UFW deny all incoming, allow only port 22 from bastion IP, 80/443 from LB
5. Disable unused services: avahi, cups, bluetooth
6. Set file descriptor limits: 65536 open files (critical for high-connection services)
7. Install Datadog agent for immediate monitoring
8. Configure log rotation

Running `ansible-playbook server-hardening.yml` applies all of this in 5 minutes. Every server is identically hardened - no configuration drift, no 'snowflake' servers."

---

## ROUND 3: SCENARIO SPEED ROUND (Minutes 35-50)

---

### Q9. [QUICK]: "GKE nodes at 90% CPU. What do you do?"

"Check Cluster Autoscaler status - why isn't it scaling? Quota hit? Scaling delay? If CA is stuck, manually scale: `gcloud container clusters resize CLUSTER --num-nodes=N`. Check for Pending pods waiting for nodes and if HPA is at maxReplicas."

---

### Q10. [QUICK]: "ArgoCD shows a service as Degraded. Your checklist?"

"1. Open ArgoCD UI, click app, find the red resource. 2. Click red resource for events and YAML diff. 3. Common causes: wrong image tag, missing imagePullSecret, resource quota exceeded, CRD version mismatch, Helm template error. 4. Fix in Git, commit, ArgoCD syncs, verify green."

---

### Q11. [QUICK]: "Datadog SLO shows 80% error budget consumed in week 2. What do you do?"

"1. Alert engineering leadership and service team immediately. 2. Enforce error budget policy - freeze non-critical deployments to this service. 3. Audit last 2 weeks - which incidents and deployments consumed the budget? 4. Prioritize reliability work: fix top error causes, improve canary strategy, add pre-production load testing."

---

### Q12. [QUICK]: "Developer says Kubernetes keeps restarting their container. How do you help?"

```bash
kubectl describe pod <name> -n <ns>    # Check 'Last State' and restart reason
kubectl logs <pod> --previous          # Logs from BEFORE the restart
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -20
```
"Common causes: OOMKilled (limit too low), failed liveness probe (wrong path/timeout), application crash (check previous logs), image pull failure."

---

### Q13. [QUICK]: "How does service discovery work in Kubernetes?"

"CoreDNS provides DNS-based service discovery. Every Service gets a DNS record automatically. Within same namespace: `service-name` resolves to ClusterIP. Across namespaces: `service-name.namespace.svc.cluster.local`. kube-proxy programs iptables rules to load-balance ClusterIP traffic to actual pod IPs. Pods just call `http://payment-api/checkout` - no external DNS needed."

---

### Q14. [QUICK]: "What is Helm and why use it over plain YAML?"

"Helm templates Kubernetes YAML. Instead of separate manifests for dev/staging/prod (with minor differences), you write one chart with variables in values.yaml per environment. Helm also manages releases: rollback, history, idempotent upgrades. At Vendasta, every microservice uses our standard Helm chart template - probes, resources, ServiceMonitor, HPA, PodDisruptionBudget all included by default."

---

*Next: File 02 - Mixed Scenario Round Part 2 (Cloud + IaC + Monitoring deep mix)*
