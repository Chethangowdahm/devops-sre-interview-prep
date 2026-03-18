# Kubernetes - Interview Questions & Answers
> Senior DevOps/SRE Level | 6+ Years Experience

---

## SECTION 1: Architecture & Fundamentals

### Q1: Explain the Kubernetes control plane components and what happens when you run "kubectl apply -f deployment.yaml"

**Answer:**

The request flow is:
1. `kubectl` sends HTTP request to **kube-apiserver** (authenticated + authorized via RBAC + admission controllers)
2. API server validates the manifest, stores it in **etcd**
3. **kube-controller-manager** (Deployment controller) detects desired state change, creates/updates ReplicaSet
4. ReplicaSet controller creates Pod objects in etcd
5. **kube-scheduler** detects unbound pods, scores nodes, assigns each pod to a node (writes to etcd)
6. **kubelet** on the selected node watches for pods assigned to it, calls CRI (containerd) to pull image and run containers
7. kubelet reports pod status back to API server → etcd

**Key insight**: Everything is event-driven. No component talks directly to another — all communication goes through etcd via the API server.

---

### Q2: What is etcd and why is it critical? How do you back it up?

**Answer:**

etcd is a distributed, consistent key-value store using the **Raft consensus algorithm**. It stores:
- All Kubernetes objects (pods, services, configmaps, secrets)
- Cluster state
- Scheduler/controller election state

**Why critical**: Losing etcd = losing all cluster state. etcd runs as a cluster (typically 3 or 5 nodes for HA) to tolerate failures.

**Backup approach:**
```bash
# Snapshot backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status snapshot.db

# Restore
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-restored
```

**Production practice**: Automate snapshots via CronJob, store in S3/GCS with retention policy.

---

### Q3: Difference between Deployment, StatefulSet, DaemonSet, and ReplicaSet?

**Answer:**

| Feature | Deployment | StatefulSet | DaemonSet | ReplicaSet |
|---|---|---|---|---|
| Pod identity | Random names | Stable (pod-0, pod-1) | Node-based | Random |
| Storage | Shared PVC or none | Per-pod PVC | Per-node | Shared or none |
| Scaling | Any order | Ordered (0→n) | One per node | Any order |
| Use case | Stateless apps | DBs, Kafka, ZK | Log agents, monitoring | Rarely direct |
| Rolling update | Yes | Yes (ordered) | Yes | No native |

**Real example**: We run MySQL via StatefulSet (pod-0 is primary, pod-1/pod-2 are replicas). We use Fluentd as DaemonSet for log shipping.

---

### Q4: How does Kubernetes networking work? Explain pod-to-pod, pod-to-service communication.

**Answer:**

**Kubernetes Networking Model**:
1. Every Pod gets a unique IP (no NAT between pods)
2. All pods can communicate with all other pods without NAT
3. Nodes can communicate with all pods without NAT

**Pod-to-Pod (same node)**:
- Pods connect via virtual ethernet pairs (veth) to a bridge (cbr0/cni0)
- Direct communication on the node network

**Pod-to-Pod (different nodes)**:
- CNI plugin (Calico/Cilium/Flannel) establishes overlay network or BGP routing
- Calico uses BGP to advertise pod CIDRs — no overlay, better performance
- Flannel uses VXLAN overlay

**Pod-to-Service**:
- Services have a virtual ClusterIP (doesn't exist as a real interface)
- kube-proxy programs iptables/IPVS rules on every node
- When pod sends traffic to ClusterIP, iptables/IPVS intercepts and load balances to a backend pod IP (DNAT)
- IPVS is preferred at scale (hashed lookup vs linear iptables scan)

---

### Q5: What happens when a pod gets OOMKilled? How do you investigate and fix it?

**Answer:**

**OOMKilled** = container exceeded its memory **limit**. Kernel sends SIGKILL.

**Investigation steps**:
```bash
# Check pod status and events
kubectl describe pod <pod-name> -n <ns>
# Look for: "OOMKilled", "Exit Code: 137"

# Check previous container logs
kubectl logs <pod-name> --previous -n <ns>

# Check metrics
kubectl top pod <pod-name> -n <ns>

# Check container memory limits
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'
```

**Root causes**:
1. Memory limit set too low for the workload
2. Memory leak in application
3. Sudden traffic spike

**Fix**:
1. Immediate: Increase memory limit in deployment
2. Short-term: Set up VPA to auto-adjust
3. Long-term: Profile the app, fix leaks, right-size with historical metrics

---

### Q6: Explain Kubernetes RBAC. How do you give a CI/CD system access to deploy?

**Answer:**

RBAC has 4 objects: Role, ClusterRole, RoleBinding, ClusterRoleBinding.

**For CI/CD deployment access**:
```yaml
# 1. Create ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production

---
# 2. Create Role with minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
# 3. Bind Role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: production
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**Best practice**: Principle of least privilege — only give the exact verbs needed.

---

### Q7: How does HPA work? What are its limitations?

**Answer:**

HPA watches metrics via the Metrics Server (or custom metrics via Prometheus Adapter) and adjusts replica count.

**Algorithm**:
```
desiredReplicas = ceil(currentReplicas * (currentMetricValue / desiredMetricValue))
```

**Limitations**:
1. **Cooldown period**: Default 5min down, 3min up — may not react fast enough to spikes
2. **Metrics Server lag**: Metrics collection has 15-60s delay
3. **Can't scale to 0**: Use KEDA for scale-to-zero (e.g., event-driven workers)
4. **Cold start**: New pods need time to start before serving traffic
5. **Resource requests required**: HPA on CPU/memory needs requests set

**Production tip**: Use KEDA for queue-based autoscaling (Kafka consumer lag, SQS depth).

---

### Q8: What is the difference between liveness and readiness probes? When would you use each?

**Answer:**

| Aspect | Liveness | Readiness |
|---|---|---|
| Failure action | **Restart** container | **Remove from Service endpoints** |
| Purpose | Is the app alive? | Is the app ready to serve? |
| Recovery | Automatic restart | Automatic re-add when healthy |

**Use Cases**:
- **Liveness**: HTTP server stuck in deadlock but not crashed → liveness restarts it
- **Readiness**: App starting up, loading cache → readiness keeps traffic away until ready
- **Startup probe**: App takes 2 minutes to start → disables liveness/readiness during startup

**Common mistake**: Setting liveness probe same as readiness — if your app takes 2min to init and liveness starts at 30s, it will restart in a loop.

---

### Q9: How do you do zero-downtime deployments?

**Answer:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # One extra pod at a time
    maxUnavailable: 0  # Never reduce below desired replicas
```

**Full checklist for zero-downtime**:
1. **maxUnavailable: 0** — never remove old pod before new one is ready
2. **Readiness probe** — new pod must pass readiness before old pod removed
3. **preStop hook + sleep** — handle in-flight requests gracefully
4. **terminationGracePeriodSeconds** — enough time for graceful shutdown

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]  # Drain in-flight requests
terminationGracePeriodSeconds: 60
```

5. **PodDisruptionBudget** — prevent too many pods from being unavailable simultaneously

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

### Q10: How do you troubleshoot a pod stuck in "CrashLoopBackOff"?

**Answer:**

CrashLoopBackOff = container starts, crashes, K8s restarts with exponential backoff (10s, 20s, 40s, 80s, 160s, 300s).

**Diagnosis steps**:
```bash
# 1. Check pod events
kubectl describe pod <pod> -n <ns>

# 2. Get logs from crashed container
kubectl logs <pod> -n <ns> --previous

# 3. Check container exit code
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
# Exit 1 = app error, Exit 137 = OOMKilled, Exit 139 = Segfault

# 4. Debug: override the command to keep container running
kubectl debug <pod> --copy-to=debug-pod --container=myapp -- sleep 3600
```

**Common causes**:
- Missing environment variables / secrets
- Wrong database connection string
- Application crash on startup (bad config)
- OOMKilled (memory limit too low)
- Missing required files/volumes

---

### Q11: What is a PodDisruptionBudget and when do you use it?

**Answer:**

PDB ensures a minimum number of pods remain available during voluntary disruptions (node drain, cluster upgrades).

```yaml
spec:
  minAvailable: 2     # At least 2 pods always running
  # OR
  maxUnavailable: 1   # At most 1 pod down at a time
```

**When to use**:
- Before cluster upgrades (node drains)
- When you have stateful workloads
- Critical services where availability must be guaranteed

**Without PDB**: During node drain, all pods can be evicted simultaneously → downtime.

---

### Q12: How do you manage secrets in Kubernetes securely?

**Answer:**

Default Kubernetes Secrets are base64-encoded (NOT encrypted) in etcd — not secure by default!

**Production approaches**:

1. **Encryption at rest**: Enable etcd encryption
```yaml
# EncryptionConfiguration
resources:
- resources: [secrets]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-32-byte-key>
  - identity: {}
```

2. **External Secret Managers** (preferred in production):
   - **AWS Secrets Manager** + External Secrets Operator
   - **HashiCorp Vault** + Vault Agent/CSI Driver
   - **GCP Secret Manager** + External Secrets Operator

3. **Sealed Secrets** (Bitnami): Encrypt secrets in Git, decrypt only in cluster

4. **IRSA/Workload Identity**: For AWS/GCP — pods get cloud IAM roles, no secret storage needed

**Best practice at Vendasta**: Use GCP Secret Manager with External Secrets Operator. Secrets never live in Git.

---

## SECTION 2: Production Scenarios

### Q13: Your cluster is out of resources and pods are stuck in Pending. What do you do?

**Answer:**

```bash
# 1. Check pending pods
kubectl get pods --all-namespaces | grep Pending

# 2. Describe a pending pod — look at "Events"
kubectl describe pod <pending-pod>
# "0/3 nodes are available: 3 Insufficient cpu"
# "0/3 nodes are available: 3 node(s) had taint... that the pod didn't tolerate"

# 3. Check node capacity
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl top nodes
```

**Actions**:
- **Node resource pressure**: Scale node group (if Cluster Autoscaler isn't working, check its logs)
- **Taints**: Add tolerations to pods or remove unnecessary taints
- **Resource quotas exceeded**: Check namespace quotas
- **Node affinity too strict**: Relax affinity rules
- **Cluster Autoscaler not scaling**: Check CA logs, verify ASG limits, check spot instance availability

---

### Q14: How do you handle a Kubernetes cluster upgrade?

**Answer:**

**Strategy** (for managed clusters like EKS/GKE):

1. **Pre-upgrade checks**:
   - Review deprecation notices for your target version
   - Test in staging first
   - Ensure PDBs are in place for critical services
   - Backup etcd

2. **Upgrade control plane** (EKS/GKE handles this — choose maintenance window)

3. **Upgrade worker nodes** (rolling node replacement):
   ```bash
   # Option A: Rolling node group replacement
   # Create new node group with new version
   # Cordon old nodes
   kubectl cordon <old-node>
   # Drain (respects PDBs)
   kubectl drain <old-node> --ignore-daemonsets --delete-emptydir-data
   # Delete old node group after pods migrate
   ```

4. **Verify**: Check all pods running, no errors in logs

5. **Rollback plan**: Keep old node group available for 24h before deleting

---

## SECTION 3: Advanced Topics

### Q15: What is CNI? Which have you used and what are the tradeoffs?

**Answer:**

CNI (Container Network Interface) is a specification for how network plugins should configure networking for pods.

| CNI | Mode | Performance | Features |
|---|---|---|---|
| **Flannel** | VXLAN overlay | Medium | Simple, limited policy |
| **Calico** | BGP (native) | High | Full NetworkPolicy, eBPF option |
| **Cilium** | eBPF | Very High | L7 policy, observability, Hubble |
| **Weave** | VXLAN/sleeve | Medium | Simple, multicast |
| **AWS VPC CNI** | Native VPC IPs | Very High | Pod IPs = VPC IPs, no overlay |

**Production experience**: Used Calico on GKE for NetworkPolicy enforcement. Used AWS VPC CNI on EKS for native routing. Evaluating Cilium for eBPF-based observability.

---

### Q16: What is the difference between ConfigMap and Secret? What are their limitations?

**Answer:**

| Aspect | ConfigMap | Secret |
|---|---|---|
| Encoding | Plain text | base64 |
| Data type | Non-sensitive config | Sensitive data |
| Size limit | **1MB** | **1MB** |
| etcd encryption | Usually not | Recommended |

**Limitations of both**:
1. **1MB limit**: Can't store large files/certs
2. **No versioning**: No built-in rotation or history
3. **No secret rotation**: Must update secret + restart pods
4. **Mounted secrets not auto-refreshed**: Env vars require pod restart; volume mounts refresh ~1min

**Solution**: External Secret Managers + External Secrets Operator for rotation and auditability.
