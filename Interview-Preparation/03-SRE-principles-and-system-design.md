# SRE Principles & System Design
> Senior SRE Level | Complete Interview Preparation

---

## PART 1: SRE CORE PRINCIPLES

### What is SRE?
Site Reliability Engineering (SRE) is what happens when you ask a software engineer to design an operations function. Invented at Google.

**Core idea**: Apply software engineering principles to operations problems.

### Key SRE Concepts

#### Toil
Manual, repetitive, automatable work that scales with service load.
SREs should spend <50% time on toil. Rest: engineering work that reduces future toil.

**Example toil**: Manual restarts, manual capacity changes, manual alerts acknowledgment.
**Solution**: Automate toil, build self-healing systems.

#### Error Budgets
Amount of unreliability allowed before SLO is breached.
**99.9% SLO**: 43.2 min/month error budget
**99.99% SLO**: 4.3 min/month error budget

**Policy**: If error budget exhausted → freeze non-reliability features. Focus engineering on reliability.

#### SLI, SLO, SLA
- **SLI**: Measurable indicator. E.g., 99.5% of requests succeed.
- **SLO**: Target. E.g., 99.9% availability.
- **SLA**: Business agreement with consequences if breached.

**Good SLIs**:
- Availability (success rate)
- Latency (p50, p99)
- Saturation (how close to capacity)
- Freshness (how old is the data)

#### Eliminating Toil via Automation
Every time you do something manually, ask:
1. Can this be automated?
2. Can the system self-heal?
3. Can this be prevented by better design?

---

## PART 2: INCIDENT MANAGEMENT

### Incident Response Process
```
1. DETECT: Alert fires (PagerDuty, phone)
2. TRIAGE: How severe? How many users affected?
   - P1/SEV1: Complete outage, all users
   - P2/SEV2: Major degradation
   - P3/SEV3: Minor issue
3. COMMUNICATE: Notify stakeholders (status page, Slack)
4. MITIGATE: Restore service (rollback, scale up, failover)
   NOTE: Mitigate first, root-cause later!
5. RESOLVE: Service fully restored
6. POST-MORTEM: Learn and prevent
```

### MTTR vs MTBF
- **MTBF** (Mean Time Between Failures): How long before next failure? → Reliability
- **MTTR** (Mean Time To Recover): How long to fix? → Recovery speed

**Focus**: Both matter. Design for fast recovery, not just low failure rate.

### Blameless Post-Mortems
Key principle: No blame. Systems failed, not people.

**Structure**:
- Date, duration, impact (users affected, revenue, SLO impact)
- Root cause
- Timeline of events
- What went well
- What went poorly
- Action items with owners and dates

**Goal**: Learn and prevent. One good post-mortem prevents 100 future incidents.

---

## PART 3: SYSTEM DESIGN FOR SREs

### Design a Highly Available Service

**Principles**:
1. **No single point of failure**: Every component redundant
2. **Graceful degradation**: Partial failure = degraded experience, not outage
3. **Bulkheads**: Isolate failures between components
4. **Circuit breakers**: Stop calling failing services
5. **Timeouts everywhere**: Never wait indefinitely
6. **Retries with backoff**: But careful — can amplify load
7. **Rate limiting**: Protect from overload

### Design: URL Shortener (system design example)

```
Requirements:
- Create short URLs
- Redirect to original URL
- Scale: 100M URLs, 1B redirects/day

Architecture:
  Client → Global LB → Web Servers (stateless, K8s, HPA)
                ↓
  Redis Cache (hot URLs, 100ms TTL)
                ↓
  PostgreSQL (original URL storage, read replicas)

Key decisions:
- Hash-based URL generation (6 chars = 56 billion URLs)
- Redis cache for 99% of redirects (most popular 10% URLs = 90% traffic)
- DB read replicas for scale
- CDN at edge for most popular URLs

SLO: 99.99% availability, p99 redirect < 20ms
```

### Design: Multi-Region Deployment

```
Requirements:
- Active-active: Serve from nearest region
- DR: Survive one region failure
- RTO: < 5 minutes, RPO: < 30 seconds

Architecture:
  Global DNS (Route53/Cloud DNS) → Route to nearest region

  Region 1 (primary):                Region 2 (secondary):
  GKE Cluster                         GKE Cluster
  Cloud SQL (primary)  ←sync→         Cloud SQL (replica)
  Redis (primary)      ←sync→         Redis (replica)

Failover:
- Cloud SQL promotion (replica → primary): ~30 seconds
- DNS TTL: 60 seconds
- Health checks trigger automatic failover

Data consistency: Last-write-wins or conflict resolution needed
```

---

## PART 4: COMMON SRE INTERVIEW QUESTIONS

### Q: How do you prioritize reliability work?
Use the error budget as a forcing function:
- Error budget healthy (>50%): Feature development has priority
- Error budget < 50%: Reliability work gets resourcing
- Budget exhausted: All hands on reliability, feature freeze

Also track 'reliability debt' — known fragile areas. Prioritize highest-risk debt.

### Q: How do you prevent outages?
1. **Change management**: Small, frequent deployments. Easy rollback.
2. **Testing**: Load testing, chaos engineering, canary deployments
3. **Capacity planning**: Proactive scaling before you need it
4. **Dependency management**: Know your dependencies' SLOs
5. **Runbooks**: Documented responses for known failure modes
6. **On-call training**: New engineers shadow before taking on-call

### Q: How do you measure reliability?
Primary metric: **SLO compliance**. Are we meeting our availability/latency targets?

Supporting metrics:
- MTTR: How fast do we recover?
- Deployment frequency: How often do we deploy (risk)?
- Change failure rate: % of deployments causing incidents
- Error budget: How much reliability 'budget' remains?

These align with **DORA metrics** (standard in SRE community):
- Deployment Frequency
- Lead Time for Changes
- Change Failure Rate
- Time to Restore Service

### Q: Describe your on-call process
"At Vendasta:
1. **Alerting**: Prometheus/Datadog alerts → Alertmanager → PagerDuty → phone call
2. **On-call rotation**: Weekly rotation, ~2-3 SREs
3. **Escalation**: If primary on-call doesn't acknowledge in 5min → secondary
4. **During incident**: Incident commander coordinates, separate investigator + communicator roles
5. **Post-incident**: Post-mortem within 48 hours for P1/P2
6. **Runbooks**: Linked from alert annotations. Step-by-step diagnosis.

**On-call hygiene**: We track alert fatigue — if >30% alerts are non-actionable, we fix the alert.
Rule: Every alert must be actionable, urgent, and have a clear response."

### Q: How do you handle capacity planning?
1. **Understand baseline**: Current resource usage patterns (CPU, memory, requests/sec)
2. **Model growth**: Business projections → expected load increase
3. **Load test**: Verify system handles 2x peak load
4. **Autoscaling**: HPA/KEDA for dynamic demand. Cluster Autoscaler for node scaling.
5. **Alerts before crisis**: Alert at 70% capacity, not 100%
6. **Review regularly**: Monthly capacity review. Quarterly strategic plan.

### Q: What's your approach to a new service launching?
SRE readiness checklist for new service:

**Reliability**
- Readiness/liveness probes configured
- Graceful shutdown (preStop hook)
- Resource requests/limits set
- PodDisruptionBudget created
- HPA configured

**Observability**
- Health metrics instrumented (/metrics endpoint)
- ServiceMonitor created
- Grafana dashboard (RED method)
- Alerting rules deployed (error rate, latency, restarts)
- Runbook linked from alerts

**Security**
- Non-root container
- Minimal RBAC (ServiceAccount with least privilege)
- Secrets via External Secrets Operator
- Network policy defined

**Operations**
- Deployment runbook documented
- On-call trained on service
- Load tested to 2x peak

---

## PART 5: QUICK REVIEW - KEY NUMBERS

Memorize these for interviews:

| SLO | Error Budget | Downtime/month |
|---|---|---|
| 99% | 1% | 7.2 hours |
| 99.9% | 0.1% | 43.2 minutes |
| 99.95% | 0.05% | 21.6 minutes |
| 99.99% | 0.01% | 4.3 minutes |
| 99.999% | 0.001% | 26 seconds |

| K8s Default | Value |
|---|---|
| Node heartbeat interval | 10s |
| Node NotReady grace period | 40s |
| Pod eviction timeout | 5 minutes |
| Default pod SIGTERM period | 30s |
| Prometheus scrape interval | 15s |
| etcd recommended nodes | 3 or 5 (odd) |
