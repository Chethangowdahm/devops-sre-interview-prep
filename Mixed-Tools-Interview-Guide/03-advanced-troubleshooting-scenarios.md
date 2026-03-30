# Mixed Tools Interview Guide - Part 3: Advanced Troubleshooting Scenarios

> Pure scenario-based questions: Production incidents involving multiple tools simultaneously.
> Each scenario has a detailed step-by-step answer with real commands and decision points.

---

## SCENARIO 1: The Cascading Failure

**[INTERVIEWER]:** "At 3 AM you get woken up by multiple alerts: payment service is down, user service is slow, and your GKE cluster is showing unusual CPU patterns. How do you approach this?"

**[TOOLS INVOLVED: Kubernetes + Datadog + Docker + Prometheus + Networking]**

**YOUR ANSWER:**
"Multiple alerts firing simultaneously usually means one root cause causing a cascade, not three independent problems. My approach:

**Step 1: Get the big picture (minute 0-3)**
```bash
# What is the cluster state overall?
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# Any recent events?
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
```

**Step 2: Look for the common thread**
In Datadog, I open the service map. If both payment and user service are degraded, what do they share? Common dependencies: same database? Same Redis cluster? Same external API? Same network path? Same node pool?

**Step 3: Check the infrastructure layer**
```bash
kubectl top nodes
# If one node is at 100% CPU and others are normal -> that node is the problem
kubectl describe node <high-cpu-node>
# Check: is this node running pods from multiple services?

# Check for node-level issues
kubectl get events --field-selector involvedObject.kind=Node
```

**Step 4: Check CoreDNS (often overlooked)**
```bash
kubectl top pods -n kube-system | grep coredns
# CoreDNS at high CPU causes ALL services to slow down (DNS timeouts)
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```
If CoreDNS is struggling, every DNS lookup in the cluster takes 5 seconds (default timeout). This causes slow responses across ALL services simultaneously.

**Step 5: Identify the blast radius**
Once root cause identified (e.g., a single node running out of resources because one pod started consuming everything):
```bash
# Find the noisy pod
kubectl top pods -A --sort-by=cpu | head -10

# Cordon the node so no new pods schedule there
kubectl cordon <problem-node>

# Evict the bad pod
kubectl delete pod <culprit-pod> -n <namespace>

# The pod reschedules to a healthy node, crisis abates
```

**Resolution:** Services recover as the noisy pod moves away from the saturated node. CoreDNS (if affected) stabilizes. Dependent services recover automatically.

**Post-mortem action items:**
- Add CPU limits to the culprit service (it had no limits)
- Set up Datadog alert on pod CPU > 90% of limit for 5+ minutes
- Implement LimitRange in each namespace to enforce defaults

**At Vendasta:** We had this exact scenario. A batch job with no resource limits consumed an entire node's CPU during its run, causing CoreDNS to compete for resources. Three services looked like they were failing independently. The fix was a LimitRange policy that I now apply to all namespaces by default."

---

## SCENARIO 2: The Mysterious Memory Leak in Production

**[INTERVIEWER]:** "Your Datadog shows a Go service's memory steadily increasing by 50MB every hour. Pods restart every 6 hours due to OOMKill. How do you find and fix the leak without causing downtime?"

**[TOOLS INVOLVED: Docker + Kubernetes + Datadog + Linux/Go profiling]**

**YOUR ANSWER:**
"Memory leak investigation requires a structured approach. Here is my exact process:

**Step 1: Confirm it's a real leak (not just growing cache)**
```promql
# Is memory growing even during low traffic?
container_memory_working_set_bytes{container='payment-api'}
# Plot this over 24 hours - if it grows even at 3 AM with no traffic, it's a leak
```

**Step 2: Take a heap profile from a live pod WITHOUT killing it**
```bash
# The service must have pprof enabled (it should in all our services)
kubectl port-forward pod/<pod-name> 6060:6060 -n production

# In another terminal:
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof -http=:8080 heap.prof
# Opens a web UI showing memory allocation by function
```

**Step 3: Take multiple profiles over time to find what's growing**
```bash
# Take heap profile every 10 minutes
for i in 1 2 3 4 5; do
  curl http://localhost:6060/debug/pprof/heap > heap_${i}.prof
  sleep 600
done

# Compare first and last
go tool pprof -base=heap_1.prof heap_5.prof
# Shows the DIFF - what grew between the two profiles
```

**Step 4: Common Go memory leak patterns to look for**

Pattern 1 - Goroutine leak:
```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | head -50
# If goroutine count grows over time -> goroutine leak
# Look for goroutines stuck in 'select' or 'chan receive' - likely a channel never closed
```

Pattern 2 - Unbounded cache/map:
```go
// BAD: This grows forever if keys are always unique
var cache = make(map[string]Response)
func handleRequest(key string, resp Response) {
    cache[key] = resp  // Never evicted!
}

// FIX: Use LRU cache with max size
import lru "github.com/hashicorp/golang-lru"
cache, _ := lru.New(10000)  // Max 10000 entries
```

Pattern 3 - HTTP response body not closed:
```go
// BAD: Memory leak - body not fully read and closed
resp, err := http.Get(url)
if err != nil { return err }
// Missing: defer resp.Body.Close()

// GOOD:
resp, err := http.Get(url)
if err != nil { return err }
defer resp.Body.Close()
body, _ := io.ReadAll(resp.Body)
```

**Step 5: Fix without downtime**
Once the leak is identified:
1. Fix in staging, run load test to confirm memory stabilizes
2. Deploy via canary: 10% of traffic to new version
3. Watch memory graph for 1 hour - if new version's memory is stable, full rollout
4. Old pods gradually replaced by rolling update

**At Vendasta:** Found a goroutine leak in our webhook processor. A goroutine was started for each incoming webhook but if the downstream service timed out, the goroutine blocked on a channel send forever. Fix: added context cancellation and a select with timeout. Goroutine count went from 50,000 (leak state) to stable 200."

---

## SCENARIO 3: Database is the Bottleneck - Full Investigation

**[INTERVIEWER]:** "Your service's p99 latency suddenly jumped from 100ms to 2 seconds. Datadog APM shows 90% of time is spent in a database query. The query runs on RDS PostgreSQL. What do you do?"

**[TOOLS TESTED: AWS RDS + PostgreSQL + Datadog + Kubernetes]**

**YOUR ANSWER:**
"Database latency issues have several causes. Here is my systematic investigation:

**Step 1: Identify the exact slow query**
```sql
-- Enable pg_stat_statements extension first
SELECT query, calls, mean_exec_time, max_exec_time, total_exec_time,
       stddev_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

**Step 2: EXPLAIN ANALYZE the slow query**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
AND o.created_at > NOW() - INTERVAL '24 hours'
ORDER BY o.created_at DESC;

-- CRITICAL: Look for these in the output:
-- Seq Scan (full table scan) -> missing index
-- Hash Join with large row counts -> might need index or query rewrite
-- Buffers: read=50000 -> lots of disk I/O, not in cache
-- Actual rows >> Estimated rows -> stale statistics, run ANALYZE
```

**Step 3: Common causes and fixes**

Cause A - Missing index:
```sql
-- Add index without locking the table
CREATE INDEX CONCURRENTLY idx_orders_status_created
ON orders(status, created_at DESC)
WHERE status = 'pending';  -- Partial index - only indexes pending orders
```

Cause B - Stale statistics causing bad query plan:
```sql
-- Update statistics for better query planning
ANALYZE orders;
ANALYZE users;

-- Reset cached plans
SELECT pg_stat_reset();
```

Cause C - N+1 query problem:
```go
// BAD: 1 query to get 100 orders + 100 queries to get each user = 101 queries
orders := db.Query('SELECT * FROM orders LIMIT 100')
for _, order := range orders {
    user := db.Query('SELECT * FROM users WHERE id = ?', order.UserID)
}

// GOOD: 1 query with JOIN
orders := db.Query(`
    SELECT o.*, u.email FROM orders o
    JOIN users u ON o.user_id = u.id
    LIMIT 100
`)
```

Cause D - Lock contention:
```sql
-- Check for blocked queries
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle'
AND now() - pg_stat_activity.query_start > interval '5 seconds'
ORDER BY duration DESC;

-- Find the blocking query
SELECT blocking_locks.pid AS blocking_pid,
       blocked_locks.pid AS blocked_pid,
       blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**Step 4: Check RDS CloudWatch metrics**
- ReadIOPS / WriteIOPS: Is disk I/O maxed out?
- FreeStorageSpace: Is the disk full?
- DatabaseConnections: Is connection pool exhausted?
- CPUUtilization: Is RDS CPU pegged?

**Step 5: If it's a traffic spike causing load**
- Add a read replica and route read queries there
- Use PgBouncer for connection pooling (each K8s pod opens connections, can overwhelm RDS)
- Scale RDS instance vertically (immediate, but causes brief downtime)

**At Altruist:** We had a sudden query slowdown after a table grew to 10M rows. The query was previously fast with a full table scan (it was only 100K rows before). EXPLAIN ANALYZE showed it needed a composite index on (user_id, status, created_at). CONCURRENTLY index creation took 8 minutes, zero downtime."

---

## SCENARIO 4: Deployment Gone Wrong

**[INTERVIEWER]:** "You deploy a new version of your service via ArgoCD. Within 5 minutes, Datadog shows error rate jumping from 0.1% to 15%. The deployment is still in progress (rolling update). What do you do?"

**[TOOLS INVOLVED: ArgoCD + Kubernetes + Datadog + Docker]**

**YOUR ANSWER:**
"This is a time-critical scenario. The rolling update means new and old pods are BOTH serving traffic simultaneously. Act fast:

**Step 1: Stop the bleeding - immediately rollback in ArgoCD**
```
1. Open ArgoCD UI
2. Click the affected application
3. Click 'History and Rollback'
4. Find the last known-good version
5. Click 'Rollback' -> Confirm
```

ArgoCD starts replacing new pods with old pods immediately. Error rate should start dropping within 2 minutes.

**Step 2: While rollback is running - understand the failure**
```bash
# Get logs from the new (bad) pods
kubectl get pods -n production | grep payment-api
# Identify which pods are 'new' (shorter AGE)

kubectl logs <new-pod-name> -n production --tail=100
# Look for: panic, error, connection refused, nil pointer

# Check events for the new pods
kubectl describe pod <new-pod> -n production | tail -30
```

**Step 3: Verify rollback worked**
```bash
kubectl rollout status deployment/payment-api -n production
# Watch: deployment 'payment-api' successfully rolled out

kubectl get pods -n production | grep payment-api
# All pods should show the OLD image tag
```

**Step 4: Communicate**
"Post in #incidents: 'Rolled back payment-api v2.3.1 due to error rate spike. Error rate returning to baseline. Investigating root cause.'"  

**Step 5: Root cause analysis (after rollback)**
- Compare the two Docker images: `docker diff` or review the actual code change
- What changed? Was it a config change, a new dependency, a code logic change?
- Reproduce in staging with load test
- Fix forward (don't just redeploy v2.3.1 - fix the actual bug)

**Prevention:** Canary deployment would have caught this at 10% traffic before it hit 100%. Error rate spike at 10% canary -> automatic rollback via AnalysisTemplate. This is why we use ArgoCD Rollouts for all Tier-1 services.

**At Vendasta:** After exactly this incident, I implemented canary rollouts with Datadog analysis. The AnalysisTemplate checks error rate at each weight step. If >1% errors, rollback is automatic - no human needed at 3 AM."

---

## SCENARIO 5: CI/CD Pipeline Security Incident

**[INTERVIEWER]:** "You notice your GitHub Actions pipeline deployed a Docker image to production that was NOT built by your CI system. Someone seems to have pushed directly to the registry. How do you detect this, respond, and prevent it?"

**[TOOLS INVOLVED: GitHub Actions + Docker + GCP Artifact Registry + Kubernetes + Security]**

**YOUR ANSWER:**
"This is a supply chain security incident. Treat it as P0 - potential compromise.

**Step 1: Confirm the incident**
```bash
# Check the image in Artifact Registry - when was it pushed? By which principal?
gcloud artifacts docker images list \
  gcr.io/vendasta-prod/payment-api \
  --include-tags

# Check Cloud Audit Logs for registry push events
gcloud logging read '
  resource.type=gcs_bucket
  logName=cloudaudit.googleapis.com/activity
  protoPayload.methodName=storage.objects.create
' --limit=50

# Does the image SHA in the running pod match what CI built?
kubectl get pod <pod> -n production -o jsonpath='{.status.containerStatuses[0].imageID}'
# Compare with the SHA from your CI pipeline logs
```

**Step 2: Immediate response**
1. Take the running pod offline if image is unverified - scale deployment to 0
2. Rotate any credentials that might have been used to push the image
3. Revoke all service account keys associated with the registry
4. Alert security team immediately

**Step 3: Forensics**
```bash
# Who pushed this image?
gcloud logging read '
  protoPayload.resourceName=projects/vendasta-prod/locations/us/repositories/containers/packages/payment-api
' \
  --format='table(timestamp, protoPayload.authenticationInfo.principalEmail)' \
  --limit=20

# Inspect the unauthorized image
docker pull gcr.io/vendasta-prod/payment-api:suspect-tag
docker inspect gcr.io/vendasta-prod/payment-api:suspect-tag
docker history gcr.io/vendasta-prod/payment-api:suspect-tag
```

**Step 4: Prevention measures I would implement**

1. **Image signing with Cosign:**
```bash
# In CI: sign the image after push
cosign sign --key cosign.key gcr.io/vendasta-prod/payment-api:abc1234

# In Kubernetes: use Kyverno policy to only allow signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-image
    match:
      resources:
        kinds: [Pod]
    verifyImages:
    - image: gcr.io/vendasta-prod/*
      key: cosign.pub
```

2. **OIDC-only registry access:** Remove all service account keys from Artifact Registry permissions. Only allow pushes from GitHub Actions via Workload Identity Federation (OIDC). No human or service account key can push directly.

3. **Registry audit alerts:** Set up Cloud Monitoring alert for any push to the production registry that does NOT have the `github-actions` service account in the audit log.

**At Vendasta:** We had a security review finding that a developer still had direct registry push access from their local machine (inherited from early days). I migrated all registry access to OIDC-only GitHub Actions and added image signing. The registry now physically cannot accept unsigned images."

---

## SCENARIO 6: Infrastructure Drift - Terraform and Reality Don't Match

**[INTERVIEWER]:** "You run `terraform plan` and see 50+ unexpected changes even though nobody touched the Terraform code. What happened and how do you handle it?"

**[TOOLS INVOLVED: Terraform + GCP/AWS]**

**YOUR ANSWER:**
"50 unexpected changes means infrastructure has drifted from the Terraform definition. This happens when someone made manual changes via the console or CLI.

**Step 1: Understand the changes before doing anything**
```bash
terraform plan -out=tfplan.bin
terraform show tfplan.bin | grep -E '^  [+-~]' | head -50

# Categorize changes:
# - Simple attribute changes (tags, descriptions) -> likely safe to apply
# - Resource recreation (- then +) -> DANGEROUS - means destroy and recreate
# - New resources -> might be intentional manual additions
```

**Step 2: Identify who made manual changes**
```bash
# GCP Cloud Audit Logs
gcloud logging read '
  resource.type=gce_instance
  logName=cloudaudit.googleapis.com/activity
  timestamp>="2024-01-01T00:00:00Z"
' --format='table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.methodName)'

# AWS CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com \
  --start-time 2024-01-01 \
  --query 'Events[].{Time:EventTime,User:Username,Event:EventName}'
```

**Step 3: Decision on how to handle each type of drift**

Case A - Manual change was intentional and correct (e.g., someone urgently added a firewall rule):
```bash
# Import the existing resource into Terraform state so Terraform manages it
terraform import google_compute_firewall.new_rule projects/vendasta-prod/global/firewalls/emergency-rule
# Now update the Terraform code to match what was manually created
# Verify with plan - should show no changes for that resource
```

Case B - Manual change was wrong (drift from desired state):
```bash
# Apply Terraform to restore the desired state
terraform apply -target=<specific-resource>  # Only the drifted resource
```

Case C - Unsure if the manual change should stay:
- Don't apply yet
- Find who made the change (audit logs)
- Get their reasoning
- Then decide: import into Terraform or revert

**Step 4: Prevent future drift**
- Set up scheduled `terraform plan` in CI that alerts if unexpected changes are found
- Remove human access to make direct changes in production (use Terraform-only)
- Use Cloud Asset Inventory (GCP) or AWS Config to detect out-of-band changes in real-time

**At Vendasta:** We discovered drift when a security engineer added emergency firewall rules during an incident directly in the GCP console and forgot to update Terraform. The next `terraform plan` showed those rules being removed. We used `terraform import` to bring them into state and documented the rules in code."

---

*Next: File 04 - Cross-Tool Architecture Questions*
