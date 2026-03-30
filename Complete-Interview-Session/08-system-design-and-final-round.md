# Complete Interview Session — File 8: System Design & Final Round

> **Round Type:** System Design + Final Technical + Closing Questions
> **Duration:** 45-60 minutes — this is where senior engineers stand out.

---

## SECTION 1: SYSTEM DESIGN

---

### Q1. Design a CI/CD pipeline for a microservices app on Kubernetes.

**[YOUR ANSWER]:**

"I design this in 4 layers:

**Layer 1 — Source Control (GitHub):**
- Feature branches → PRs → Code Review → Merge to main
- Branch protection: required reviews + passing CI checks before merge

**Layer 2 — CI (GitHub Actions):**
```
On PR:
  1. Lint + static analysis
  2. Unit tests + code coverage
  3. Security scan (Snyk/Trivy)
  4. Build Docker image (multi-stage)
  5. Scan image for vulnerabilities

On merge to main:
  1. All PR checks
  2. Push image to GCR with git SHA tag
  3. Update image tag in manifest repo
```

**Layer 3 — CD (ArgoCD GitOps):**
```
Manifest repo change detected:
  1. ArgoCD auto-syncs to staging
  2. Integration tests run
  3. Manual approval gate for production
  4. ArgoCD Rollout: Canary 10% → 50% → 100%
  5. Datadog monitors error rate — auto-rollback if > 1%
```

**Layer 4 — Observability:**
- Build/deploy notifications in Slack
- Deployment events in Datadog
- Post-deploy SLO check

**Key design decisions:**
- Separate CI and CD manifest repos — clean separation
- No pipeline has cluster credentials — ArgoCD pulls
- Canary + automated analysis — deployment failures caught early

**At Vendasta:** This exact pipeline. Went from 2 deploys/week to 10+/day safely."

---

### Q2. Design a highly available web application on GCP.

**[YOUR ANSWER]:**

"For 99.9% SLO with multi-zone redundancy:

```
[Users]
    |
[Cloud CDN]           Static assets, DDoS protection
    |
[Global Load Balancer] SSL termination, health checks
    |
[GKE Cluster - 3 zones]
  Pod  Pod  Pod        Kubernetes Deployment, HPA
    |
[Cloud SQL Multi-AZ]  Primary + Read Replica, auto-failover
    |
[Redis (Memorystore)]  Session cache, rate limiting
    |
[GCS]                  Object storage
```

**Redundancy decisions:**
- Pods spread across 3 zones via PodAntiAffinity
- minReplicas: 3 (at least 1 per zone)
- PodDisruptionBudget: maxUnavailable: 1
- Cloud SQL multi-AZ failover (~30 seconds)

**Zero-downtime deployment:**
- RollingUpdate: maxUnavailable: 0, maxSurge: 1
- Readiness probes — no traffic until fully ready
- Graceful shutdown: SIGTERM handler + 30s grace period

**At Vendasta:** During a GCP zone outage, our service was unaffected. GKE rescheduled pods in remaining 2 zones automatically."

---

### Q3. How would you design observability for 100 microservices?

**[YOUR ANSWER]:**

Observability at scale must be standardized and automated.

**Architecture:**
```
Microservice → OpenTelemetry SDK
                  |
         Traces + Metrics + Logs
                  |
      Tempo + Prometheus + Loki
                  |
            Grafana (unified)
                  |
           Alertmanager → PagerDuty/Slack
```

**Key principles:**
1. Standardize with OpenTelemetry — single SDK, vendor-agnostic
2. Auto-instrumentation — no code changes for basic tracing
3. Consistent naming — service names, environment labels, team labels enforced via Helm chart
4. Sampling: 100% for errors, 10% for normal requests
5. SLO-driven alerting — all tier-1 services have SLOs

**At Vendasta:** Standardized on OpenTelemetry + Datadog. New services onboard in < 1 hour — Helm chart includes all DD_ env vars automatically."

---

## SECTION 2: ADVANCED SCENARIOS

---

### Q4. Your company wants to migrate from monolith to microservices. What is your approach?

**[YOUR ANSWER]:**

I use the Strangler Fig pattern — incrementally extract services, never rewrite everything at once.

**Phase 1 — Containerize the monolith:**
- Dockerize, run on Kubernetes
- Set up CI/CD and observability first — can't measure improvement without it

**Phase 2 — Extract high-value, low-complexity services:**
- Find bounded contexts — authentication, notifications
- Route traffic to new service via API gateway or feature flags

**Phase 3 — Incremental carve-out:**
- Each service has its own database — no shared DB
- Use Pub/Sub or Kafka for async communication
- Monolith shrinks until core functionality remains

**Risks:** Distributed transactions (use Saga pattern), network latency (measure service-to-service), observability complexity (tracing is critical).

**At Altruist:** Extracted authentication from Django monolith. Feature flags routed 1% → 10% → 100% traffic over 2 weeks with instant rollback."

---

### Q5. How do you handle database migrations with zero downtime?

**[YOUR ANSWER]:**

Use the Expand/Contract pattern — never do breaking changes directly.

```
Step 1 — EXPAND: Add new column with default (keep old)
Step 2 — BACKFILL: Populate new column in batches
Step 3 — DEPLOY: App writes to both old and new columns
Step 4 — CONTRACT: Switch reads to new column only
Step 5 — CLEANUP: Drop old column in next window
```

**Example:**
```sql
-- Bad: breaks live app immediately
ALTER TABLE users RENAME COLUMN email TO email_address;

-- Good: expand-contract
-- Step 1: add new column
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);
-- Step 2: backfill in batches (no full table lock)
UPDATE users SET email_address = email WHERE id BETWEEN 1 AND 10000;
-- Step 3: deploy dual-write app, then switch reads, then drop old
```

**At Vendasta:** All Cloud SQL migrations tested against nightly prod snapshot in staging. Zero failed migrations in 18 months using this pattern."

---

## SECTION 3: FINAL QUESTIONS

---

### Q6. What trends in DevOps/SRE excite you most?

**[YOUR ANSWER]:**

**Platform Engineering:** Internal Developer Platforms (IDP) — Backstage by Spotify — abstract K8s/CI/CD complexity. Scale SRE knowledge to 100+ developers without proportional headcount.

**eBPF for Observability:** Tools like Cilium, Pixie, Tetragon provide kernel-level observability without app code changes. Revolutionary for legacy systems.

**GitOps Maturity:** Beyond ArgoCD for deployments — applying GitOps to networking (Cilium), security policies (OPA Gatekeeper), and infrastructure.

**AI-assisted operations:** Datadog Watchdog, anomaly detection, GitHub Copilot for infra code. Genuinely reducing toil."

---

### Q7. Hardest technical problem you've solved?

**[STAR METHOD]:**

**Situation:** At Vendasta, intermittent p99 latency spikes to 8 seconds in checkout — only for 5% of requests, couldn't reproduce.

**Task:** Previous team investigated for 2 weeks without finding root cause. I was asked to solve it.

**Action:** Switched approach from metrics to APM traces. Filtered by duration > 5s. Found 4-5 second gaps in traces — no spans during that time. Not a slow query — a connection wait. Investigated Cloud SQL connection pool: max was 100 DB connections, but 15 pods x 10 connections each = 150 demand. Implemented PgBouncer to pool 150 app connections to 30 DB connections.

**Result:** p99 dropped from 8s (spikes) to 180ms consistently. Key insight: trace gaps are as informative as slow spans. The team had been looking at the wrong layer entirely."

---

## SECTION 4: QUICK REFERENCE — SRE NUMBERS

| SLO | Monthly Downtime | Weekly Downtime |
|-----|------------------|-----------------|
| 99% | 7.31 hours | 1.68 hours |
| 99.9% | 43.8 minutes | 10.1 minutes |
| 99.95% | 21.9 minutes | 5 minutes |
| 99.99% | 4.38 minutes | 1 minute |

| Kubernetes | Key Numbers |
|------------|-------------|
| etcd quorum (3 nodes) | 2 nodes needed for writes |
| Pod start latency (image pulled) | 5-10 seconds |
| GKE node upgrade window | ~5 min per node with PDB |
| HPA default scale-up cooldown | 15 seconds |
| HPA default scale-down cooldown | 5 minutes |

---

## SECTION 5: SMART QUESTIONS TO ASK THE INTERVIEWER

Pick 2-3 — shows senior thinking and genuine interest:

1. 'How do you balance feature velocity with reliability? Do you have an error budget policy?'
2. 'What is the biggest infrastructure or reliability challenge the team is currently facing?'
3. 'What does the growth path look like from Senior SRE here?'
4. 'What does the on-call rotation look like — how many pages per week typically?'
5. 'What parts of the stack are you most excited to modernize in the next 12 months?'
6. 'How do SRE and development teams collaborate here — embedded, centralized, or consulting model?'

---

## FINAL CHECKLIST — DAY OF INTERVIEW

**Morning before:**
- [ ] Review your 5 STAR stories (incidents resolved, migrations led, improvements made)
- [ ] Research the company's tech stack from JD
- [ ] Write down 3 questions to ask
- [ ] Reminder: 6.2 years of production experience — I am ready

**During interview:**
- [ ] Pause before answering — 10 seconds of thinking is okay
- [ ] Anchor to real experience: 'At Vendasta, what we found was...'
- [ ] Show tradeoffs: 'The alternative would be X, but we chose Y because...'
- [ ] In system design: ask scope questions first — scale, requirements, constraints

**If you don't know something:**
'I haven't used [X] directly, but based on my experience with [similar tool], here's how I'd approach it...'

---

**Chethan — You have 6.2 years of real production experience across Vendasta, Altruist, and Ascer. You've managed GKE clusters serving 60,000+ businesses, implemented GitOps, built observability stacks, and handled production incidents. You are more prepared than 95% of candidates. Trust your experience. Go get it!**
