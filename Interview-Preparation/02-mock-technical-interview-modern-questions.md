# Mock Technical Interview - What Interviewers Ask in 2025/2026
> Senior DevOps/SRE | Complete Q&A with Model Answers
> Format: Questions asked as an interviewer would ask them

---

## ROUND 1: SCREENING / PHONE SCREEN

### Q: Walk me through how your CI/CD pipeline works at your current company.

**Model Answer:**
"At Vendasta, we have a fully automated pipeline:
1. Developer pushes code to GitHub — triggers GitHub Actions or Jenkins CI
2. CI runs: build, unit tests, integration tests, Docker image build
3. Docker image pushed to GCR (Google Container Registry) tagged with git SHA
4. CI updates the image tag in our GitOps repository (separate repo)
5. ArgoCD detects the change in GitOps repo and automatically syncs to the target cluster
6. ArgoCD does rolling update — new pods come up, pass readiness probes, old pods go down
7. Slack notification on success/failure

For production: we have a promotion gate — staging must be healthy for 30 minutes before auto-promoting to production."

---

### Q: How do you ensure zero-downtime deployments?

**Model Answer:**
"Several things need to be in place:
1. **RollingUpdate strategy** with maxUnavailable=0, maxSurge=1
2. **Readiness probe** — new pod must be healthy before old pod is removed
3. **preStop hook + sleep** — drain in-flight requests before container stops (sleep 10-15s)
4. **terminationGracePeriodSeconds** — enough time for graceful shutdown
5. **PodDisruptionBudget** — prevents all pods from going down at once

The most common mistake is forgetting the preStop hook — Kubernetes removes the pod from the service immediately when it starts terminating, but in-flight requests can still be reaching it for a few seconds."

---

### Q: What is the difference between IaC and configuration management?

**Model Answer:**
"IaC (Infrastructure as Code) provisions and manages cloud resources — creating VPCs, EC2 instances, GKE clusters, IAM policies. Terraform is the prime example. It manages the infrastructure lifecycle.

Configuration Management configures and maintains the state of servers and applications — installing packages, managing config files, setting up users. Ansible is the most common example.

They complement each other: Terraform creates the EC2 instance, Ansible configures what runs on it. In a Kubernetes world, the distinction blurs — containers are immutable so you don't configure running containers. IaC manages the cluster, and container images replace configuration management for apps."

---

## ROUND 2: TECHNICAL DEEP DIVE

### Q: How would you design a highly available infrastructure on GCP for a SaaS application with 100,000 users?

**Model Answer:**
"Here's how I'd approach it:

**Compute**: GKE Autopilot cluster in a regional configuration (us-central1) — spans 3 AZs automatically. All microservices as Deployments with pod anti-affinity to spread across zones.

**Networking**: Global HTTP(S) Load Balancer at the edge. Cloud CDN for static assets. Cloud Armor for DDoS/WAF protection. VPC with separate subnets per concern (app, DB, tooling).

**Database**: Cloud SQL with HA (primary + standby in different zones). Read replicas for read-heavy workloads. Connection pooling via PgBouncer.

**Cache**: Memorystore (Redis) for session data and hot cache. Regional with replication.

**Reliability mechanisms**:
- PodDisruptionBudgets for all critical services
- HPA with KEDA for autoscaling
- Circuit breakers (via Istio or app-level)
- Health checks at all layers
- Multi-region failover for DR (RPO/RTO defined by business)

**Observability**: Datadog for full-stack monitoring, Prometheus/Grafana for K8s metrics, Cloud Logging for centralized logs, SLOs defined per service.

**Security**: Workload Identity (no static credentials), Secret Manager for secrets, VPC Service Controls, least-privilege IAM."

---

### Q: A service is experiencing intermittent high latency. Walk me through your investigation.

**Model Answer:**
"This is systematic — I'd follow the signals:

**Step 1: Define scope**
- Is it all users or specific ones? One endpoint or all? Started at a specific time?

**Step 2: Check metrics (Datadog/Prometheus)**
- Request latency: p50, p95, p99 — if p99 is high but p50 is fine, it's a tail latency issue
- Error rate alongside latency
- CPU/memory/disk on pods and nodes

**Step 3: Distributed tracing**
- Find slow traces in Datadog APM
- Flame graph: where is time being spent? DB? External API? Serialization?

**Step 4: Correlate with events**
- Did latency start after a deployment? After traffic spike?
- Check ArgoCD deployment history

**Step 5: Common causes to check**
- DB slow queries (check slow query log, missing indexes)
- GC pauses (if JVM) — check GC metrics
- Network issues (packet loss, retransmits)
- Noisy neighbor (resource contention on node)
- Connection pool exhaustion

**Step 6: Fix and verify**
- If DB: add index, optimize query
- If GC: increase heap, change GC algorithm
- If noisy neighbor: move pod to different node, set resource limits"

---

### Q: How do you handle secrets in a Kubernetes/GitOps environment?

**Model Answer:**
"This is a common pain point. My approach at Vendasta:

**Rule #1**: Secrets NEVER go in Git — not even encrypted base64.

**Our solution**: External Secrets Operator + GCP Secret Manager.
- Secrets are created and rotated in GCP Secret Manager
- ExternalSecret CR in Git references the secret (by name, not value)
- External Secrets Operator syncs it to a K8s Secret at runtime
- Pods reference the K8s Secret normally

**Benefits**:
- Secrets are version-controlled and audited in GCP Secret Manager
- Rotation happens without redeploying (ESO refreshes on schedule)
- Access control via GCP IAM (only specific service accounts can read secrets)
- No secrets in Git = no accidental exposure

**For CI/CD secrets**: Use OIDC/Workload Identity Federation so GitHub Actions assumes a GCP SA directly — no static credentials stored in GitHub."

---

### Q: Explain what happens when a Kubernetes node fails

**Model Answer:**
"When a node fails:

1. **kubelet stops heartbeating** to API server (every 10s by default)
2. **Node controller** marks node as NotReady after 40s (node-monitor-grace-period)
3. After 5 minutes (pod-eviction-timeout) of NotReady, node controller **evicts pods** and marks them for rescheduling
4. **Scheduler** places evicted pods on other nodes
5. If Cluster Autoscaler is running, it may **provision a new node** if needed

**What doesn't self-heal without help**:
- StatefulSet pods without stable storage (need PV reattachment)
- Pods with `hostPath` volumes (data is on the dead node)
- PodDisruptionBudget violations (if too many pods are on the failed node, may not reschedule all)

**Our runbook**: Alert on NodeNotReady after 2 minutes. Check if the node is truly dead (SSH test). If dead, drain and delete it. Cluster Autoscaler will replace."

---

### Q: What is service mesh and when would you use it?

**Model Answer:**
"A service mesh adds a sidecar proxy (Envoy) to every pod that intercepts all network traffic.

**What it gives you**:
- **mTLS**: Mutual TLS between all services — automatic encryption + authentication
- **Observability**: Traffic metrics, distributed tracing, service maps — without code changes
- **Traffic management**: Canary deployments, A/B testing, circuit breakers, retries
- **Authorization policies**: Control which services can call which

**Popular options**: Istio, Linkerd (lighter weight), Consul Connect

**When to use**: Large microservice environments (20+ services) where you need zero-trust networking, detailed observability, or sophisticated traffic routing.

**When NOT to use**: Small teams, early-stage product — the operational complexity is significant.

**Our approach at Vendasta**: We haven't adopted a full service mesh yet — we use Datadog APM for tracing and GKE's built-in NetworkPolicy for security. Service mesh is on the roadmap."

---

### Q: How do you implement SLOs in practice?

**Model Answer:**
"SLOs (Service Level Objectives) are the foundation of SRE. Here's our practical approach:

**Step 1: Define SLIs (what to measure)**
- Availability: percentage of successful requests
- Latency: p99 < 200ms
- Error rate: < 0.1%

**Step 2: Set SLOs**
- 99.9% availability = 43.2 min downtime/month budget

**Step 3: Implement measurement (PromQL)**
```promql
# Availability SLI
sum(rate(http_requests_total{status!~'5..'}[5m]))
/ sum(rate(http_requests_total[5m]))
```

**Step 4: Alert on error budget burn rate**
- 1-hour burn rate > 14x = page
- 6-hour burn rate > 6x = ticket

**Step 5: Dashboards**
- Grafana SLO dashboard: current burn rate, remaining budget, burn rate history

**Step 6: Error budget policy**
- Budget > 50% remaining: feature development OK
- Budget < 50%: increase reliability investment
- Budget exhausted: freeze releases until budget recovers"

---

## ROUND 3: MODERN TOPICS (2025/2026)

### Q: What's your experience with GitOps and how does it compare to traditional CI/CD?

**Model Answer:**
"I've been implementing GitOps with ArgoCD at Vendasta for the past 1.5 years.

**Traditional CI/CD** (push model): CI pipeline builds image, then directly deploys to cluster via kubectl/helm. Pipeline has deploy credentials.

**GitOps** (pull model): Git is the source of truth. ArgoCD constantly reconciles cluster with Git. No deploy credentials in CI.

**Key benefits I've experienced**:
1. **Audit trail**: Every deployment is a Git commit. Who deployed what, when, why (PR description)
2. **Easy rollback**: Git revert = immediate rollback, with history
3. **Self-healing**: ArgoCD reverts manual cluster changes automatically (drift detection)
4. **Reduced blast radius**: CI pipeline can't directly touch production — it only updates Git
5. **Developer autonomy**: Teams manage their own deployment configs as code

**Result**: Reduced deployment errors by 70% at Vendasta."

---

### Q: How do you approach chaos engineering?

**Model Answer:**
"Chaos engineering is deliberately injecting failures to validate system resilience.

**Our approach**:
1. Start with hypothesis: 'System will maintain 99.9% availability when one AZ goes down'
2. Define steady state: normal error rate, latency, request throughput
3. Inject failure in staging first: terminate pods, kill nodes, introduce network latency
4. Observe: does the system self-heal? Are alerts firing?
5. Analyze and fix gaps
6. Gradually run in production (small blast radius, easy rollback)

**Tools**: Chaos Monkey (Netflix), LitmusChaos (K8s-native), Gremlin (commercial)

**Simple chaos we run**: Regularly kill random pods in staging. This validates pod restart, rescheduling, and readiness probes work correctly.

**Not implemented yet** at Vendasta: Network partitions, AZ-level failures. That's on our roadmap."

---

### Q: What's your approach to cost optimization in Kubernetes/cloud?

**Model Answer:**
"Cost optimization is an ongoing practice, not a one-time project.

**Kubernetes cost levers**:
1. **Right-size resource requests**: Use VPA recommendations + actual usage metrics to set right CPU/memory requests. Oversized requests waste node capacity.
2. **HPA/KEDA**: Scale to demand — don't overprovision for peak traffic
3. **Scale to zero**: KEDA for queue workers — zero pods when no work
4. **Spot/Preemptible nodes**: 60-90% cheaper. Use for fault-tolerant workloads.
5. **Node auto-provisioning**: Cluster Autoscaler removes idle nodes

**GCP cost levers**:
- Committed Use Discounts for baseline capacity
- GCS Lifecycle policies (move old objects to cheaper tiers)
- VPC endpoints for GCS (avoid egress costs)

**Tools**: Kubecost for K8s cost visibility, GCP Cost Management dashboards.

**Real impact at Vendasta**: Reviewed all service resource requests vs actual usage. Found most services had 3-5x overprovisioned CPU requests. Rightsizing saved ~30% on compute costs."

---

### Q: Explain your experience with Platform Engineering

**Model Answer:**
"Platform Engineering is about building internal developer platforms (IDP) that let development teams deploy, monitor, and manage their services without deep ops knowledge.

**At Vendasta, I worked on platform enablement**:
- **Self-service**: Created Helm charts and deployment templates so devs can deploy new services without SRE involvement
- **GitOps templates**: Standard ArgoCD ApplicationSet templates for new services
- **Observability self-service**: ServiceMonitor templates, standard Grafana dashboards for new services
- **Developer documentation**: Runbooks, how-to guides for common operations

**Goal**: SRE team focuses on reliability infrastructure; developers can ship independently.

**Modern tools**: Backstage (developer portal), Crossplane (K8s-based infrastructure provisioning), ArgoCD ApplicationSets."

---

## COMMON TRICK QUESTIONS

### Q: What is the difference between Docker and a VM?
Containers share the host OS kernel — they're isolated processes using Linux namespaces and cgroups. No hypervisor.
VMs have their own OS kernel, run on hypervisor (full hardware emulation).
Containers: MBs, ms startup, near-zero overhead.
VMs: GBs, minutes startup, 5-20% overhead.

### Q: What happens when you delete a deployment in Kubernetes?
1. Deployment controller marks all ReplicaSet replicas as desired=0
2. ReplicaSet controller sends SIGTERM to pods
3. Pods run preStop hooks, wait terminationGracePeriodSeconds
4. SIGKILL sent to any remaining processes
5. Pod objects removed from etcd
6. ReplicaSet object deleted
7. Deployment object deleted
Note: PersistentVolumeClaims and Secrets are NOT deleted — they have separate lifecycle.

### Q: Can Kubernetes pods communicate across namespaces?
By default YES — all pods can communicate regardless of namespace (flat network model).
To restrict: Use NetworkPolicies. They're additive whitelisting — by default all traffic is allowed, NetworkPolicy adds restrictions.

### Q: What is the CAP theorem and how does it apply to etcd?
CAP = Consistency, Availability, Partition tolerance. Pick 2.
etcd is CP: Strongly consistent (all reads reflect latest write), partition tolerant.
Sacrifices: If network partition, etcd may become unavailable rather than return stale data.
This is why etcd needs quorum (majority of nodes). 3 nodes: can lose 1. 5 nodes: can lose 2.

---

## CLOSING TIPS

1. **Structure your answers**: Problem → Analysis → Solution → Result
2. **Use the STAR method for behavioral**: Situation, Task, Action, Result
3. **Think aloud**: For system design, narrate your thinking
4. **Admit what you don't know**: 'I haven't used X directly, but based on Y I'd approach it like this'
5. **Ask clarifying questions**: Especially for system design — understand scale, constraints
6. **Show depth + breadth**: Mention alternatives, tradeoffs — shows senior thinking
7. **Reference real experience**: 'At Vendasta we solved this by...' — specific > generic
