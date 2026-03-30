# Complete Interview Session — File 3: Kubernetes & Docker Deep Dive

> **Round Type:** Core technical round — most SRE/DevOps interviews spend 30–45 min here.
> **Your Edge:** You manage GKE clusters at Vendasta serving 60,000+ businesses. Speak from production experience.

---

## SECTION 1: KUBERNETES ARCHITECTURE

---

### Q1. "Explain the Kubernetes architecture. What happens when you run kubectl apply?"

**[YOUR ANSWER]:**
"Kubernetes has two planes:

**Control Plane:**
- **kube-apiserver:** The front door — all kubectl commands, controllers, and kubelets talk to it. It validates, authenticates, and stores state in etcd.
- **etcd:** The distributed key-value store. The single source of truth for ALL cluster state.
- **kube-scheduler:** Watches for unscheduled pods, selects the best node based on resources, affinity rules, taints/tolerations.
- **kube-controller-manager:** Runs reconciliation loops — Deployment controller maintains replicas, Node controller handles node failures.
- **cloud-controller-manager:** Integrates with cloud APIs — creates Load Balancers, manages node lifecycles on GKE/EKS.

**Worker Node:**
- **kubelet:** Agent on each node — talks to API server, ensures containers defined in PodSpecs are running.
- **kube-proxy:** Manages iptables/IPVS rules for Service networking.
- **Container runtime:** containerd or CRI-O — actually pulls and runs containers.

**What happens on kubectl apply:**
1. kubectl sends the manifest to kube-apiserver via REST
2. API server authenticates (cert), authorizes (RBAC), validates (admission controllers)
3. Resource stored in etcd
4. Deployment controller detects delta — creates ReplicaSet
5. ReplicaSet controller creates Pods
6. Scheduler assigns Pods to nodes
7. Kubelet on target node pulls image, starts container
8. Container is running, kubelet reports status back to API server"

---

### Q2. "What is etcd and what happens if it goes down?"

**[YOUR ANSWER]:**
"etcd is a distributed, consistent key-value store based on the Raft consensus algorithm. It stores all Kubernetes cluster state — pods, services, configs, secrets, RBAC rules.

**If etcd goes down:**
- Existing workloads continue running — kubelets and containers are independent
- NO new changes can be made — kubectl commands fail
- Auto-healing stops — if a pod dies, it won't be recreated
- Essentially, the cluster is read-only from a management perspective

**Production practice:**
- etcd should have an odd number of members (3 or 5) for Raft quorum
- Regular etcd backups are critical: `etcdctl snapshot save backup.db`
- On GKE, Google manages etcd — but on self-managed clusters, I always set up automated etcd backups to GCS

**Raft quorum:** In a 3-node etcd cluster, you need 2 nodes (majority) alive for writes. Losing 2 of 3 nodes means the cluster is read-only until quorum is restored."

---

### Q3. "Explain the difference between a Deployment, StatefulSet, and DaemonSet."

**[YOUR ANSWER]:**

**Deployment:**
- For stateless applications
- Pods are interchangeable — any pod can serve any request
- Supports rolling updates and rollbacks
- Example: web servers, REST APIs, microservices

**StatefulSet:**
- For stateful applications with persistent identity
- Pods have stable network names (pod-0, pod-1) and persistent storage
- Ordered startup/shutdown
- Example: databases (Postgres, Kafka, Zookeeper, Elasticsearch)

**DaemonSet:**
- Runs exactly ONE pod on every node (or subset of nodes)
- Used for cluster-level agents that need to run everywhere
- Example: Datadog agent, Fluentd log collector, node-exporter, CNI plugins

**At Vendasta:** Our microservices run as Deployments. Datadog agent runs as a DaemonSet so we get metrics from every GKE node. Kafka runs as a StatefulSet with persistent volumes."

---

### Q4. "A pod is stuck in Pending state. How do you debug it?"

**[SCENARIO — Very common interview question]**

**[YOUR ANSWER]:**
"Pending means the pod is not scheduled onto any node. I debug systematically:

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look at the Events section at the bottom
```

**Common causes and fixes:**

1. **Insufficient resources:**
   - Event: `0/5 nodes are available: 5 Insufficient cpu`
   - Fix: Scale up node pool or reduce resource requests

2. **No nodes match node selector or affinity:**
   - Event: `0/5 nodes are available: 5 node(s) didn't match node selector`
   - Fix: Check `nodeSelector` and `nodeAffinity` in pod spec

3. **Taints and tolerations:**
   - Event: `0/5 nodes are available: 5 node(s) had taint {key: value}`,
   - Fix: Add matching toleration to pod spec

4. **PVC not bound:**
   - Event: `persistentvolumeclaim not found` or `waiting for PVC to be bound`
   - Fix: Check if StorageClass exists, PVC is in Bound state

5. **Image pull issue (ImagePullBackOff):**
   - Event: `Failed to pull image: access denied`
   - Fix: Check imagePullSecrets, verify registry credentials

**At Vendasta:** We once had a Pending pod because the GKE node pool had a taint for GPU nodes and our service accidentally had a node selector pointing to those nodes. `kubectl describe pod` revealed the taint mismatch immediately."

---

### Q5. "What is a liveness probe vs readiness probe vs startup probe?"

**[YOUR ANSWER]:**

**Liveness Probe:** Answers 'Is this container alive?'
- If it fails, kubelet kills and restarts the container
- Use for detecting deadlocks, stuck processes

**Readiness Probe:** Answers 'Is this container ready to serve traffic?'
- If it fails, pod is removed from Service endpoints (no traffic sent)
- Container keeps running — it's just not receiving traffic
- Use for warm-up time, dependency checks

**Startup Probe:** For slow-starting containers
- Disables liveness and readiness checks until startup probe succeeds
- Prevents premature restarts during initialization

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30   # 30 * 10s = 5 min to start
  periodSeconds: 10
```

**At Vendasta:** Every microservice has all three probes. A missing readiness probe caused a major incident — a new pod received traffic before it had loaded its config from ConfigMap, resulting in 500 errors. After that, we made probes mandatory in our Helm chart template."

---

### Q6. "How does Kubernetes networking work? Explain pod-to-pod communication."

**[YOUR ANSWER]:**
"Kubernetes networking follows four fundamental rules:
1. Every pod gets its own unique IP address
2. Pods can communicate with any other pod without NAT
3. Nodes can communicate with all pods without NAT
4. The IP a pod sees for itself is the same IP others use to reach it

**CNI (Container Network Interface):** A plugin that implements these rules. Common options: Calico, Flannel, Cilium, Weave. On GKE we use Google's VPC-native networking.

**Pod to Pod (same node):** Traffic goes through a virtual ethernet pair (veth) and a bridge (cbr0). No physical network hop.

**Pod to Pod (different nodes):** Traffic goes through the node's physical NIC. CNI handles routing — on GKE, each node has a /24 subnet allocated from VPC, so routing is native.

**Service networking:** kube-proxy programs iptables/IPVS rules that DNAT (destination NAT) traffic from ClusterIP to actual pod IPs. Services provide stable virtual IPs regardless of pod churn."

---

### Q7. "What is HPA, VPA, and KEDA? How do they differ?"

**[YOUR ANSWER]:**

**HPA (Horizontal Pod Autoscaler):**
- Scales pod count based on metrics (CPU, memory, or custom metrics)
- Adds/removes pods
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**VPA (Vertical Pod Autoscaler):**
- Adjusts CPU/memory requests and limits for existing pods
- Cannot do both HPA and VPA on same resource metric
- Use VPA in Recommendation mode first to right-size requests

**KEDA (Kubernetes Event-Driven Autoscaler):**
- Scales based on external event sources — queue depth, Kafka lag, Datadog metrics, cron
- Can scale to zero (HPA minimum is 1)
- At Vendasta, we use KEDA to scale workers based on Pub/Sub queue depth

**When to use what:**
- HPA: CPU/memory-bound web services
- VPA: batch jobs, services with unpredictable resource needs
- KEDA: event-driven workloads, queue consumers, scheduled jobs"

---

### Q8. "What are taints and tolerations? Give a real use case."

**[YOUR ANSWER]:**
"**Taints** are applied to nodes and repel pods from being scheduled there unless the pod has a matching toleration.

**Tolerations** are applied to pods and allow them to be scheduled on tainted nodes.

```bash
# Taint a node for GPU workloads only
kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule

# Pod with toleration can be scheduled there
tolerations:
- key: 'dedicated'
  operator: 'Equal'
  value: 'gpu'
  effect: 'NoSchedule'
```

**Three taint effects:**
- `NoSchedule:` New pods won't be scheduled unless they tolerate
- `PreferNoSchedule:` Scheduler tries to avoid, but not hard rule
- `NoExecute:` Existing pods are evicted unless they tolerate

**Real use case at Vendasta:**
"We taint dedicated high-memory nodes for our ML inference pods. Regular microservices don't have the toleration, so they never land on those expensive nodes. This enforces workload isolation and prevents cost overruns.""

---

## SECTION 2: DOCKER DEEP DIVE

---

### Q9. "What is the difference between COPY and ADD in Dockerfile?"

**[YOUR ANSWER]:**
"**COPY** is straightforward — it copies files from build context into the image. Preferred for most use cases.

**ADD** has two extra features: it can extract tar archives automatically, and it can fetch URLs. However, these extra features make it unpredictable and harder to reason about.

**Best practice:** Always use COPY unless you specifically need ADD's features. Even for tar extraction, many prefer to use `RUN tar -xf ...` with COPY for clarity."

---

### Q10. "How do you optimize Docker image size? Show a multi-stage build."

**[YOUR ANSWER]:**
"Multi-stage builds are the most powerful technique — you build in one stage and copy only the final artifact to a minimal runtime image.

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Stage 2: Runtime (scratch = empty image)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Result:** Build image ~350MB → Runtime image ~8MB

**Other optimization techniques:**
- Use official slim/alpine/distroless base images
- Order layers from least to most frequently changed (COPY source code LAST)
- Combine RUN commands with `&&` to reduce layers
- Use .dockerignore to exclude unnecessary files
- Never install debugging tools in production images

**At Vendasta:** We reduced our Node.js service image from 1.2GB to 180MB using multi-stage builds and distroless base, which improved pull times and reduced attack surface."

---

### Q11. "A container is crashing with OOMKilled. How do you fix it?"

**[YOUR ANSWER]:**
"OOMKilled means the container exceeded its memory limit and was killed by the Linux OOM killer.

**Diagnosis:**
```bash
kubectl describe pod <pod-name>
# Look for: Last State: Terminated, Reason: OOMKilled

kubectl top pod <pod-name>
# Current memory consumption
```

**Fix options:**
1. **Increase memory limit** (short-term fix):
```yaml
resources:
  limits:
    memory: '512Mi'   # Increase from 256Mi
  requests:
    memory: '256Mi'
```

2. **Fix the memory leak** (real fix):
   - Check application code for memory leaks
   - Add memory profiling: `go tool pprof`, `heap dumps` for JVM
   - Review if caches are unbounded

3. **Use VPA recommendations:**
   - VPA in recommendation mode analyzes historical usage
   - Right-size requests and limits accordingly

**At Vendasta:** A Node.js service had a memory leak in an event listener that was never cleaned up. The service worked fine under normal load but OOMKilled under peak traffic. We found it with `--inspect` flag and Chrome DevTools heap profiling."

---

### Q12. "What is the difference between CMD and ENTRYPOINT?"

**[YOUR ANSWER]:**
"**ENTRYPOINT** defines the executable that always runs. It cannot be overridden by `docker run` arguments (unless you use `--entrypoint` flag).

**CMD** provides default arguments to ENTRYPOINT, or the default command if no ENTRYPOINT is set. It CAN be overridden by arguments passed to `docker run`.

```dockerfile
ENTRYPOINT ["/app/server"]
CMD ["--port", "8080"]   # Default args, can be overridden

# docker run myimage --port 9090  <- overrides CMD
# Resulting command: /app/server --port 9090
```

**Best practice:** Use ENTRYPOINT for the main process, CMD for default arguments. This makes containers both runnable with defaults and configurable."

---

### Q13. "Scenario: A microservice in production is responding slowly. Walk me through your investigation."

**[SCENARIO-BASED — Tests end-to-end SRE thinking]**

**[YOUR ANSWER]:**
"I would approach this in layers, from external to internal:

**Step 1 — Confirm the scope:**
- Is it all endpoints or one specific API path?
- Is it all pods or a subset?
- When did it start? Was there a recent deployment?

**Step 2 — Check Datadog/Grafana:**
- p99 latency trend — is it gradual or sudden?
- Error rate — are there more 5xx errors?
- Trace view — where in the request chain is the bottleneck?

**Step 3 — Check Kubernetes layer:**
```bash
kubectl top pods -n production
# Is CPU/memory maxed out?

kubectl describe hpa -n production
# Is HPA already at maxReplicas?

kubectl get events -n production --sort-by='.lastTimestamp'
# Any evictions, OOMKills, scheduling delays?
```

**Step 4 — Check dependencies:**
- Is the database slow? Check DB slow query logs
- Is an external API timing out? Check connection pool exhaustion
- Is Redis latency elevated?

**Step 5 — Check infrastructure:**
- GKE node CPU/memory saturation
- Network latency between pods
- DNS resolution time (CoreDNS issues are common under high load)

**Real example at Vendasta:**
'Our checkout service became slow during a traffic spike. Traces in Datadog showed 90% of the latency was in a single DB query. The HPA had already scaled pods to max, but the query wasn't using an index. We added a composite index on (user_id, created_at) and latency dropped from 2s to 80ms immediately.'"

---

*Next: Move to File 04 — CI/CD & GitOps Round*
