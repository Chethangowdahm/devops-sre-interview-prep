# Kubernetes - Core Concepts
> For Senior DevOps/SRE Engineers (6+ Years Experience)

---

## 1. What is Kubernetes?
Kubernetes (K8s) is an open-source container orchestration platform that automates deploying, scaling, and managing containerized applications across clusters of machines. Originally designed by Google, now maintained by CNCF.

**Key principles**: Desired state management, self-healing, declarative configuration.

---

## 2. Architecture

### Control Plane
- **kube-apiserver**: Entry point for all REST calls. Validates, processes, persists to etcd.
- **etcd**: Distributed key-value store. Source of truth. Raft consensus.
- **kube-scheduler**: Assigns pods to nodes using filters (predicates) and scoring (priorities).
- **kube-controller-manager**: Node, ReplicaSet, Endpoints, ServiceAccount controllers.
- **cloud-controller-manager**: Cloud-specific: load balancers, volumes, routes.

### Worker Nodes
- **kubelet**: Node agent. Ensures containers match PodSpec via CRI.
- **kube-proxy**: Implements Service networking via iptables/IPVS.
- **Container Runtime**: containerd, CRI-O (CRI-compliant).

---

## 3. Core Objects

### Pod
Smallest schedulable unit. Containers share network namespace and storage.

### Deployment
Manages ReplicaSets. Rolling updates, rollbacks, scaling.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### Service Types
- **ClusterIP**: Internal only (default)
- **NodePort**: External via node IP:port (30000-32767)
- **LoadBalancer**: Cloud LB
- **ExternalName**: DNS alias

### StatefulSet
Stable identities (pod-0, pod-1), ordered ops, per-pod PVCs. For DBs, Kafka, ZooKeeper.

### DaemonSet
One pod per node. For log agents (Fluentd), monitoring (Node Exporter), CNI plugins.

### Job / CronJob
- **Job**: Run to completion
- **CronJob**: Scheduled (cron syntax)

---

## 4. Configuration

### ConfigMap
Non-sensitive config as key-value or files. Decoupled from image.

### Secret
Sensitive data (base64). Types: Opaque, kubernetes.io/tls, kubernetes.io/dockerconfigjson.

---

## 5. Storage

### Volume Types
| Type | Lifecycle | Use Case |
|---|---|---|
| emptyDir | Pod lifetime | Shared tmp between containers |
| hostPath | Node lifetime | DaemonSet log dirs |
| configMap/secret | External | Config injection |
| PVC | Independent | Durable app data |

### Dynamic Provisioning Flow
StorageClass → PVC created → PV auto-provisioned → Bound → Pod uses PVC

---

## 6. Networking

### CNI Rules
- Each pod gets unique IP
- All pods can communicate (no NAT)
- CNI plugins: Calico, Cilium, Flannel, Weave

### DNS
`<service>.<namespace>.svc.cluster.local`

### Network Policies
Default: all traffic allowed. NetworkPolicy is additive whitelisting.

### Ingress
HTTP/HTTPS routing. Requires Ingress Controller (Nginx, Traefik, ALB).

---

## 7. Scheduling

### Priority Order
1. **NodeSelector**: Label matching (simple)
2. **Node Affinity**: required/preferred rules (expressive)
3. **Taints/Tolerations**: Node repels pods unless tolerated
4. **Pod Affinity/Anti-Affinity**: Schedule near/away from other pods
5. **TopologySpreadConstraints**: Even distribution across zones/nodes

### Resource Requests vs Limits
- **Requests**: Scheduler uses this. Pod guaranteed these resources.
- **Limits**: Container cannot exceed. CPU throttled; memory → OOMKilled.

### QoS Classes
- **Guaranteed**: requests == limits for all containers
- **Burstable**: requests < limits
- **BestEffort**: no requests/limits (evicted first)

---

## 8. Autoscaling

### HPA (Horizontal Pod Autoscaler)
Scale pods based on CPU, memory, or custom metrics.

### VPA (Vertical Pod Autoscaler)
Adjust requests/limits automatically. Requires pod restart.

### KEDA
Event-driven autoscaling (Kafka lag, SQS depth, cron-based).

### Cluster Autoscaler
Add/remove nodes based on pending pods.

---

## 9. RBAC

| Object | Scope | Purpose |
|---|---|---|
| Role | Namespace | Define permissions |
| ClusterRole | Cluster-wide | Define permissions |
| RoleBinding | Namespace | Bind Role to subject |
| ClusterRoleBinding | Cluster-wide | Bind ClusterRole to subject |

Subjects: User, Group, ServiceAccount

---

## 10. Health Probes

| Probe | Failure Action | Use |
|---|---|---|
| **Liveness** | Restart container | Detect deadlocks/crashes |
| **Readiness** | Remove from Service endpoints | App not ready for traffic |
| **Startup** | Restart if not started | Slow-starting apps |

---

## 11. Helm

Package manager for K8s.
- **Chart**: Bundle of K8s manifests + templates
- **Release**: Chart instance running in cluster
- **Values**: Customization via values.yaml

```bash
helm install myapp ./mychart -f custom-values.yaml
helm upgrade myapp ./mychart --set image.tag=v2.0
helm rollback myapp 1
helm list -n production
```

---

## 12. Key kubectl Commands

```bash
# Debugging
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -c <container> --previous --tail=100
kubectl exec -it <pod> -- /bin/sh

# Rollouts
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2
kubectl set image deployment/<name> app=myapp:v3

# Resources
kubectl top pods --sort-by=cpu
kubectl top nodes

# Maintenance
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# Events
kubectl get events --sort-by=.lastTimestamp -n <ns>

# Dry run
kubectl apply -f manifest.yaml --dry-run=client
kubectl diff -f manifest.yaml
```
