# SRE Principles & Concepts — Interview Prep (6 YOE)

---

## What is SRE?

**Site Reliability Engineering** = Software Engineering applied to Operations problems.
Originated at Google. The SRE book defines it as:

> "SRE is what happens when you ask a software engineer to design an operations function."

Core philosophy: **Treat operations as a software engineering problem.** Automate toil, use code to manage infrastructure, measure everything, make decisions based on data.

---

## SRE vs DevOps

```
DevOps:  Cultural movement. Break silos between Dev and Ops.
         Shared responsibility, collaboration, continuous delivery.

SRE:     Specific implementation of DevOps principles.
         Concrete practices: SLOs, error budgets, toil reduction.
         Originated at Google. A job title AND a set of practices.

Both emphasize: automation, CI/CD, monitoring, fast feedback loops.
```

---

## SLI → SLO → SLA

```
SLI (Service Level Indicator):
  A quantitative measure of service behavior.
  "What are we measuring?"
  Examples: request success rate, latency P99, availability %, error rate

SLO (Service Level Objective):
  A target value for an SLI.
  "What's our internal target?"
  Examples: 99.9% of requests succeed, P99 latency < 500ms
  INTERNAL target — not a contract.

SLA (Service Level Agreement):
  A contract with customers.
  "What do we promise externally?"
  Usually weaker than SLO (SLO = 99.9%, SLA = 99.5%)
  Breach → financial penalties / credits
```

### Defining Good SLIs

```
Availability SLI:
  Good: count(successful requests) / count(total requests)
  Note: "successful" = status code NOT 5xx (server error)

Latency SLI:
  Good: count(requests under 500ms) / count(total requests)
  Better than average — averages hide tail latency

Freshness SLI (data pipelines):
  Good: % of data updates processed within SLO time window

Coverage SLI (batch jobs):
  Good: % of records processed without error

Correctness SLI:
  Good: % of results returning correct/expected output
```

### SLO Examples (real-world)

| Service | SLI | SLO |
|---------|-----|-----|
| API Gateway | Request success rate | 99.9% (43.8 min downtime/month) |
| Payments | P99 latency | < 2 seconds |
| Data pipeline | Data freshness | 95% of updates within 1 hour |
| Storage | Durability | 99.999999999% (11 nines) |

---

## Error Budget

```
Error Budget = 1 - SLO target (as % of time or requests)

For 99.9% SLO over 30 days:
  Error budget = 0.1% = 43.8 minutes

If we have 10 minutes of downtime → 10/43.8 = 22.8% budget consumed.
Remaining: 33.8 minutes.

Error Budget Policy:
  Budget > 50% remaining → ship features freely
  Budget < 50% remaining → prioritize reliability work
  Budget exhausted       → FREEZE feature releases until budget recovers
                           Focus 100% on reliability
```

### Burn Rate
```
Burn rate = how fast you're consuming error budget

1x burn rate   = budget depleted exactly at end of window (OK)
14.4x burn rate = depleted in 2 hours (CRITICAL — page now)

Multi-window burn rate alerting:
  Fast burn (1h window + 5min window): Alert if burning > 14.4x
  Slow burn (6h window + 30min window): Alert if burning > 6x
  Warning (1d or 3d window): Alert if burning > 3x
```

---

## Toil

> "Toil is the kind of work tied to running a production service that tends to be manual, repetitive, automatable, tactical, devoid of enduring value, and that scales linearly as a service grows."
> — Google SRE Book

**Examples of toil:**
- Manually restarting pods when they OOMKill
- Manually approving/running database migrations
- Copying backups manually each night
- Responding to the same false-positive alert repeatedly
- Manual capacity planning spreadsheets

**SRE goal:** Spend < 50% of time on toil. Rest on engineering work that reduces future toil.

**Toil reduction examples:**
- Write operator/controller to auto-restart and page if OOMKills > threshold
- Automate DB migrations in CI pipeline
- Tune alert (add `for: 5m` to eliminate false positives)
- Write runbook, then automate the runbook steps

---

## Reliability Engineering Practices

### Capacity Planning
```
Process:
1. Define growth model (users, requests, storage per month)
2. Collect current utilization data (Grafana/Datadog dashboards)
3. Project: if 20% user growth → X% more CPU/memory/storage
4. Identify bottlenecks before they occur
5. Provision ahead of time (node pools, read replicas, CDN)

Tools:
  - Prometheus: resource usage trends
  - GKE: node pool autoscaling limits
  - BigQuery: query historical metrics for trend analysis
```

### Chaos Engineering
```
Principle: Intentionally inject failures to find weaknesses before prod does.

Tools:
  - Chaos Monkey (Netflix): randomly kill VMs
  - Chaos Mesh / LitmusChaos: K8s chaos experiments
  - Gremlin: enterprise chaos platform

Experiments (start small!):
  1. Kill a random pod → does HPA + Service recover?
  2. Add network latency between services → do timeouts/retries work?
  3. Drain a node → does workload reschedule properly?
  4. Exhaust memory → does OOMKill + recovery work?
  5. Inject HTTP 503 from dependency → does circuit breaker trip?

Rules:
  - Always start in staging
  - Have a clear "stop" mechanism
  - Define hypothesis: "Killing 1 pod should not affect user-visible availability"
  - Measure impact on SLIs during experiment
```

### Reliability Patterns
```
Circuit Breaker:
  Track failure rate to dependency → if > threshold, "open circuit"
  (return error fast without calling failing service)
  After cooldown → try again ("half-open")

Retry with Exponential Backoff + Jitter:
  First retry: wait 1s
  Second retry: wait 2s + random(0-1s)
  Third retry: wait 4s + random(0-2s)
  Max retries: 3-5 (avoid retry storms)

Timeouts:
  Always set! (downstream service hanging = your service hangs)
  Set per-request and per-total timeout separately

Bulkhead:
  Isolate resources. Payment service gets its own thread pool/goroutine pool.
  Failure in payments doesn't exhaust all threads for user service.

Graceful Degradation:
  If recommendation service is down, show empty recommendations (not 500)
  If cache is down, fall back to DB (slower but available)
```

---

## Incident Management

### Incident Severity Levels

```
SEV1 (Critical):
  - Production is down for all users
  - Data loss or corruption
  - Security breach
  Response: Immediate, all hands, executive notification
  MTTR target: < 1 hour

SEV2 (High):
  - Significant degradation (>10% users affected)
  - Key feature unavailable
  Response: On-call + backup on-call
  MTTR target: < 4 hours

SEV3 (Medium):
  - Minor degradation, small % affected
  - Workaround available
  Response: On-call during business hours
  MTTR target: < 24 hours

SEV4 (Low):
  - Non-urgent, minimal user impact
  Response: Next business day
```

### Incident Response Playbook

```
1. DETECT
   PagerDuty alert → on-call acknowledges

2. TRIAGE
   - What's the user impact? (SEV classification)
   - Is this still happening? (check metrics)
   - Quick hypothesis: recent deploy? infrastructure change?

3. COMMUNICATE
   - Open incident Slack channel: #inc-YYYYMMDD-short-description
   - Post status page update (if user-facing)
   - Notify stakeholders for SEV1/2

4. MITIGATE (stop the bleeding first)
   - Rollback the deploy (if deploy-related)
   - Increase replicas (if capacity issue)
   - Redirect traffic (if regional issue)
   - Disable broken feature flag
   Priority: RESTORE SERVICE first, find root cause second

5. RESOLVE
   - Confirm metrics returned to normal
   - Update status page: "Resolved"
   - Hand off to follow-up (if outside work hours)

6. POST-INCIDENT (within 48 hours)
   - Write blameless postmortem
   - 5-whys root cause analysis
   - Action items: assigned, prioritized, tracked in Jira
```

### Blameless Postmortem Template

```markdown
## Incident: [Short title] — [Date]

### Impact
- Duration: X minutes
- Users affected: Y%
- Revenue impact: (if known)

### Timeline (UTC)
- 02:30 - Deploy of v1.2.3 completed
- 02:33 - Error rate began rising (automated alert)
- 02:34 - PagerDuty paged on-call
- 02:38 - Rollback initiated
- 02:41 - Error rate returned to baseline

### Root Cause
The v1.2.3 release included a change to the payment deserialization
that failed to handle the legacy payment format (pre-2023 records).
Payments older than 2023 that triggered retry logic threw a
NullPointerException, causing 500 responses.

### Contributing Factors
- No integration test covered legacy payment format
- Canary deployment was not used (would have caught in 5% traffic)

### What Went Well
- Automated alerting detected within 3 minutes
- On-call responded quickly and identified deploy correlation fast
- Rollback was quick (< 5 min)

### Action Items
| Action | Owner | Priority | Due |
|--------|-------|----------|-----|
| Add legacy payment format test | @dev-team | P1 | 2026-04-01 |
| Implement canary deployments | @sre-team | P2 | 2026-04-15 |
| Add payment format smoke test | @qa-team | P1 | 2026-04-05 |
```

---

## SRE Interview Questions — 6 YOE Level

**Q: How do you write a good SLO?**
> Start with user journeys, not infrastructure metrics. "Users can successfully submit a payment" → SLI: payment API success rate. Set target based on current baseline (if you're at 99.5%, don't immediately go to 99.99% — you'll burn all budget). SLO should be just good enough to keep users happy. Review quarterly. Document what counts as "good" vs "bad" in your SLI definition.

**Q: How do you handle alert fatigue?**
> Every alert that pages must be actionable. Remove alerts that fire more than they should. Add `for: 5m` to eliminate transient spikes. Add runbooks to every alert. Use anomaly detection for variable baselines. Group related alerts in Alertmanager. Monthly review: which alerts had the most noise? Fix or delete them. If you're consistently ignoring an alert, delete it — it's worse than no alert.

**Q: What is your approach to on-call?**
> Clear escalation path. Runbook for every alert. Track on-call burden (number of pages per shift, MTTR). If > 2 pages/shift, investigate and fix. Blameless culture — if paged at 3am, the system is the problem, not the engineer. After every incident: action items to prevent recurrence. Rotate on-call fairly. Compensate appropriately (time off or pay).

**Q: How do you reduce toil?**
> Identify: track all toil in a log for 2 weeks. Quantify: how many hours/week? Prioritize: highest-volume + highest-risk toil first. Automate: write a script/operator/job. Validate: measure hours saved. Example: We had 3 hours/week of manual log rotation. Wrote a Cloud Scheduler + Cloud Function to automate it. Reduced to 0.

**Q: What's the difference between MTTR and MTBF? How do you optimize each?**
> **MTBF** (Mean Time Between Failures): How often does it break? Improve by: better testing, canary deployments, chaos engineering, code quality. **MTTR** (Mean Time To Recover): How fast do you fix it? Improve by: better observability (fast detection), runbooks (fast diagnosis), automated rollback (fast mitigation), clear escalation (fast response). SRE focuses more on MTTR because failures are inevitable.

**Q: Design a monitoring strategy for a new microservice.**
> 1. **Four Golden Signals**: latency, traffic, errors, saturation — add these first. 2. **Health endpoint** `/healthz` and `/ready` for probes. 3. **Structured logging** (JSON) with request_id, user_id, trace_id. 4. **Distributed tracing** — instrument with Datadog APM or OpenTelemetry. 5. **SLO** — define before launch, not after. 6. **Runbook** for every alert. 7. **Dashboard** — RED metrics, SLO panel. 8. **On-call rotation** — include service owners from day 1.

**Q: How do you handle a thundering herd problem?**
> Thundering herd: many clients retry simultaneously after a service recovers → immediately overwhelms it again. Solutions: exponential backoff + jitter in clients, rate limiting at API gateway, circuit breaker (don't retry when circuit is open), gradual traffic ramp-up after recovery, cache warming before traffic acceptance (readiness probe only passes after cache warm).

---

## SRE Golden Rules

```
1. Reliability is a feature. Ship it like one.
2. Every page must be actionable. If it can wait, it shouldn't page.
3. If you automate a human action, the human should review the automation.
4. Postmortems are blameless — blame the system, fix the system.
5. Measure your SLOs. If you don't measure it, you can't improve it.
6. Toil compounds. Eliminate it before it eliminates your team.
7. Reduce the blast radius of every change.
8. The best runbook is the one that automates itself out of existence.
9. Design for failure. Everything fails eventually.
10. The SRE's job is to make the on-call rotation boring.
```
