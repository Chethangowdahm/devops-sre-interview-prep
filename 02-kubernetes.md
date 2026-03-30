# Kubernetes — Deep Dive for SRE/DevOps (6 YOE)

---

## What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform** that automates:
- Deployment, scaling, and self-healing of containerized applications
- Load balancing and service discovery
- Rolling updates and rollbacks
- Resource management across a cluster of nodes

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         CONTROL PLANE (Master)                               │
│                                                                              │
│  ┌─────────────┐  ┌────────┐  ┌────────────────┐  ┌─────────────────────┐  │
│  │  API Server │  │  etcd  │  │   Scheduler    │  │ Controller Manager  │  │
│  │  (front     │  │ (state │  │  (assign pods  │  │ (reconcile desired  │  │
│  │   door)     │  │  store)│  │   to nodes)    │  │  vs actual state)   │  │
│  └─────────────┘  └────────┘  └────────────────┘  └─────────────────────┘  │
│         ▲                                                                    │
│         │ kubectl / API calls                                                │
└─────────┼────────────────────────────────────────────────────────────────────┘
          │
┌─────────┼────────────────────────────────────────────────────────────────────┐
│  DATA PLANE (Worker Nodes)                                                   │
│         │                                                                    │
│  ┌──────▼──────────┐   ┌───────────────────┐   ┌───────────────────┐        │
│  │    Node 1       │   │     Node 2        │   │     Node 3        │        │
│  │  ┌───────────┐  │   │  ┌───────────┐    │   │  ┌───────────┐    │        │
│  │  │  kubelet  │  │   │  │  kubelet  │    │   │  │  kubelet  │    │        │
│  │  │(runs pods)│  │   │  │           │    │   │  │           │    │        │
│  │  └───────────┘  │   │  └───────────┘    │   │  └───────────┘    │        │
│  │  ┌───────────┐  │   │  ┌───────────┐    │   │  ┌───────────┐    │        │
│  │  │kube-proxy │  │   │  │kube-proxy │    │   │  │kube-proxy │    │        │
│  │  │(iptables/ │  │   │  │           │    │   │  │           │    │        │
│  │  │ ipvs)     │  │   │  └───────────┘    │   │  └───────────┘    │        │
│  │  └───────────┘  │   │  ┌───────────┐    │   │  ┌───────────┐    │        │
│  │  ┌───────────┐  │   │  │  Pod      │    │   │  │  Pod      │    │        │
│  │  │  Pod      │  │   │  │ ┌───────┐ │    │   │  │ ┌───────┐ │    │        │
│  │  │ ┌───────┐ │  │   │  │ │  C1   │ │    │   │  │ │  C1   │ │    │        │
│  │  │ │  C1   │ │  │   │  │ └───────┘ │    │   │  │ └───────┘ │    │        │
│  │  │ └───────┘ │  │   │  └───────────┘    │   │  └───────────┘    │        │
│  │  └───────────┘  │   └───────────────────┘   └───────────────────┘        │
│  └─────────────────┘                                                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|-----------|------|
| **API Server** | Single entry point. All kubectl/client calls go here. Validates and processes REST requests. |
| **etcd** | Distributed key-value store. Holds ALL cluster state. Critical — must be backed up. |
| **Scheduler** | Watches for unscheduled pods, selects best node based on resources, taints, affinity. |
| **Controller Manager** | Runs controllers: ReplicaSet, Deployment, Node, Job controllers. Ensures desired = actual state. |
| **Cloud Controller Manager** | GKE-specific: manages GCP LoadBalancers, persistent disks, node lifecycle. |

### Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Agent on each node. Ensures containers in pods are running and healthy. |
| **kube-proxy** | Maintains network rules (iptables/IPVS). Implements Service abstraction. |
| **Container Runtime** | containerd or CRI-O. Actually runs containers. |

---

## Core Kubernetes Objects

### Pod
Smallest deployable unit. One or more containers sharing network and storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: gcr.io/project/myapp:v1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: host
```

### Deployment
Manages ReplicaSets. Provides rolling updates, rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # max extra pods during update
      maxUnavailable: 0  # no downtime
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: gcr.io/project/myapp:v1.0.0
        # ... (same as pod spec above)
```

### Service
Stable network endpoint for pods (pods are ephemeral, Services are not).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP   # or LoadBalancer, NodePort
```

**Service types:**
```
ClusterIP    → Internal only. Default. Accessed within cluster.
NodePort     → Exposes on each Node's IP:Port (30000-32767). Dev/test only.
LoadBalancer → Creates GCP LoadBalancer. External access. (GKE)
ExternalName → DNS alias to external service.
```

### Ingress
HTTP/HTTPS routing rules. One LoadBalancer for multiple services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
```

### ConfigMap & Secret

```yaml
# ConfigMap (non-sensitive config)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  CONFIG_FILE: |
    [server]
    port = 8080

---
# Secret (base64 encoded — use Sealed Secrets or External Secrets in prod)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:          # auto-base64 encodes
  host: "10.90.80.234"
  password: "supersecret"
```

---

## Scaling

### Horizontal Pod Autoscaler (HPA)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Vertical Pod Autoscaler (VPA)
Adjusts CPU/memory requests automatically. Do NOT combine with HPA on same metric.

### Cluster Autoscaler (GKE)
Adds/removes nodes based on pending pods and underutilization.

---

## Pod Disruption Budget (PDB)
Ensures minimum availability during voluntary disruptions (node drain, updates).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2      # or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

---

## Resource Management

```
Requests: Guaranteed minimum. Used for scheduling decisions.
Limits:   Hard cap. Exceeding CPU → throttled. Exceeding memory → OOMKilled.

QoS Classes:
  Guaranteed  → requests == limits  (highest priority, never OOM-evicted first)
  Burstable   → requests < limits
  BestEffort  → no requests/limits  (evicted first under pressure)
```

**Rule of thumb:** Always set requests. Set limits conservatively (avoid OOMKills in prod).

---

## Storage

```
PersistentVolume (PV)      → Actual storage resource (GCP Persistent Disk)
PersistentVolumeClaim (PVC) → Request for storage by a pod
StorageClass               → Dynamic provisioning (pd-standard, pd-ssd, etc.)
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: pd-ssd
  resources:
    requests:
      storage: 20Gi
```

---

## RBAC (Role-Based Access Control)

```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRole/ClusterRoleBinding → cluster-wide scope.

---

## Key kubectl Commands

```bash
# Context management
kubectl config get-contexts
kubectl config use-context gke_project_region_cluster
kubectl config set-context --current --namespace=production

# Basic operations
kubectl get pods -n production -o wide
kubectl get pods -n production -w                    # watch
kubectl describe pod myapp-xyz -n production
kubectl logs myapp-xyz -n production -f --tail=100
kubectl exec -it myapp-xyz -n production -- sh

# Deployments
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production  # rollback
kubectl scale deployment myapp --replicas=5 -n production

# Debugging
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl top pods -n production
kubectl top nodes
kubectl debug node/node-name -it --image=ubuntu

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -k overlays/production/              # Kustomize
helm upgrade --install myapp ./chart -n production -f values.prod.yaml

# Drain node (maintenance)
kubectl cordon node-name        # stop scheduling new pods
kubectl drain node-name --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node-name      # re-enable scheduling
```

---

## Networking Deep Dive

```
Pod-to-Pod:     Direct IP communication (flat network via CNI)
Pod-to-Service: kube-proxy iptables/IPVS rules → ClusterIP
External-to-Pod: LoadBalancer → NodePort → kube-proxy → Pod

DNS (CoreDNS):
  Service DNS: my-svc.my-namespace.svc.cluster.local
  Pod DNS:     pod-ip.my-namespace.pod.cluster.local
```

**Network Policies** (firewall for pods):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

---

## Interview Questions — Kubernetes (6 YOE Level)

**Q: What happens when you run `kubectl apply -f deployment.yaml`?**
> kubectl sends the manifest to the API Server → API Server validates and writes to etcd → Controller Manager's Deployment controller detects the new desired state → creates/updates ReplicaSet → ReplicaSet controller creates Pods → Scheduler assigns Pods to Nodes → kubelet on the node pulls the image and starts containers.

**Q: What is the difference between liveness and readiness probes?**
> **Liveness**: Is the container alive? If it fails, kubelet restarts the container. **Readiness**: Is the container ready to serve traffic? If it fails, the pod is removed from Service endpoints (no traffic), but NOT restarted. Use readiness to handle startup time and graceful degradation.

**Q: How does a rolling update work, and how do you ensure zero downtime?**
> Deployment creates a new ReplicaSet, gradually scales it up while scaling down the old one. Zero downtime: set `maxUnavailable: 0`, `maxSurge: 1`, have proper readiness probes (new pods only get traffic when ready), set `terminationGracePeriodSeconds` properly, and use PodDisruptionBudget.

**Q: What is etcd and what happens if it goes down?**
> etcd is the cluster state store. If it goes down: existing pods keep running (kubelet is autonomous), but no new deployments, no scaling, no config changes can be made (API server cannot persist state). This is why etcd is run as a HA cluster (3 or 5 nodes) and must be backed up regularly.

**Q: How do you handle OOMKilled pods?**
> Check `kubectl describe pod` for OOMKilled in LastState. Check actual memory usage with `kubectl top pod`. Increase memory limit or investigate memory leak in app. Consider VPA to auto-tune. Ensure QoS is Guaranteed for critical services.

**Q: Explain the difference between StatefulSet and Deployment.**
> **Deployment**: Pods are interchangeable, no stable identity. Good for stateless apps. **StatefulSet**: Pods have stable network identity (pod-0, pod-1), stable storage (each pod gets its own PVC), ordered deployment/termination. Required for databases (MySQL, Kafka, Elasticsearch).

**Q: What are taints and tolerations?**
> **Taint** on a Node: "Don't schedule pods here unless they tolerate this." **Toleration** on a Pod: "I accept this taint." Used to dedicate nodes (e.g., GPU nodes only for ML workloads), or prevent pods from running on master nodes.

**Q: How do you debug a pod stuck in Pending state?**
```bash
kubectl describe pod <name>    # look at Events section
# Common reasons:
# - Insufficient CPU/memory on nodes → check requests vs node capacity
# - No node matches nodeSelector/affinity
# - Taint not tolerated
# - PVC not bound
# - Image pull error
kubectl get nodes               # check node status
kubectl describe node <node>    # check allocatable resources
```

---

## Pod Lifecycle — State Machine

```
                         kubectl apply / scheduler assigns pod to node
                                        │
                                        ▼
                              ┌──────────────────┐
                              │     PENDING      │
                              │ (scheduled, image │
                              │  pulling, init   │
                              │  containers)     │
                              └────────┬─────────┘
                          ┌───────────┴────────────┐
                 image pull fails              all containers
                 node out of resources         started
                          │                        │
                          ▼                        ▼
                ┌─────────────────┐      ┌──────────────────┐
                │  (still Pending)│      │     RUNNING      │
                │  ImagePullBack  │      │  (at least one   │
                │  Off/ErrImage   │      │   container up)  │
                └─────────────────┘      └────────┬─────────┘
                                          ┌───────┴───────┐
                                    all containers    container
                                    exit code 0       crashes /
                                    restartPolicy=    OOMKilled
                                    Never             │
                                          │           ▼
                                          │   ┌───────────────┐
                                          │   │  (Restarting) │
                                          │   │ CrashLoopBack │
                                          │   │ Off if > 5    │
                                          │   │ restarts      │
                                          │   └───────────────┘
                                          ▼
                              ┌──────────────────────┐
                              │      SUCCEEDED        │
                              │  (all containers      │
                              │   exited with 0)      │
                              └──────────────────────┘

                              ┌──────────────────────┐
                              │       FAILED          │
                              │  (at least one        │
                              │   container non-0     │
                              │   exit, no restart)   │
                              └──────────────────────┘

                              ┌──────────────────────┐
                              │      UNKNOWN          │
                              │  (node unreachable,   │
                              │   no heartbeat)       │
                              └──────────────────────┘
```

---

## HPA — Feedback Control Loop

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HPA CONTROL LOOP (every 15 seconds)                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  HPA Controller                                                  │    │
│  │                                                                  │    │
│  │  1. COLLECT metrics from metrics-server (or custom metrics API)  │    │
│  │     current CPU: 350m across 3 pods = 116m avg per pod          │    │
│  │                                                                  │    │
│  │  2. CALCULATE desired replicas:                                  │    │
│  │     desiredReplicas = ceil(current * currentMetric/targetMetric) │    │
│  │     = ceil(3 * 116/70) = ceil(4.97) = 5                         │    │
│  │                                                                  │    │
│  │  3. APPLY scale constraints:                                     │    │
│  │     min(max(desiredReplicas, minReplicas), maxReplicas)          │    │
│  │     = min(max(5, 2), 20) = 5                                     │    │
│  │                                                                  │    │
│  │  4. SCALE: kubectl scale deployment/myapp --replicas=5           │    │
│  │                                                                  │    │
│  │  5. COOLDOWN: wait scaleUpStabilizationWindow (default 0s)      │    │
│  │              wait scaleDownStabilizationWindow (default 5min)    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                │                           ▲                             │
│                │ scale                     │ metrics                     │
│                ▼                           │                             │
│  ┌───────────────────────┐    ┌────────────────────────────────────┐    │
│  │  Deployment           │    │  metrics-server                    │    │
│  │  replicas: 3 → 5      │    │  (aggregates kubelet stats)        │    │
│  │  ┌─────┐┌─────┐┌─────┐│   │  or                                │    │
│  │  │Pod 1││Pod 2││Pod 3││   │  Prometheus Adapter (custom metric) │    │
│  │  └─────┘└─────┘└─────┘│   │  or Datadog Cluster Agent          │    │
│  │  ┌─────┐┌─────┐       │   └────────────────────────────────────┘    │
│  │  │Pod 4││Pod 5│ NEW   │                                              │
│  │  └─────┘└─────┘       │                                              │
│  └───────────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────────┘

Scale-down protection:
  5 minutes of sustained low usage before scaling down (prevents flapping)
  Can tune: --horizontal-pod-autoscaler-downscale-stabilization=5m
```

---

## RBAC — Permission Evaluation

```
┌─────────────────────────────────────────────────────────────────────────┐
│              RBAC AUTHORIZATION FLOW                                     │
│                                                                          │
│  Request: kubectl get pods -n production                                 │
│      │                                                                   │
│      │ Who is making this request?                                       │
│      ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  AUTHENTICATION (who are you?)                                   │   │
│  │  ├─ User certificate → extracted from kubeconfig                 │   │
│  │  ├─ ServiceAccount token → pod calling API server                │   │
│  │  └─ OIDC token → cloud IAM identity                              │   │
│  └──────────────────────────────────┬───────────────────────────────┘   │
│                                     │ identity verified                  │
│                                     ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  AUTHORIZATION (are you allowed?)                                │   │
│  │                                                                  │   │
│  │  API Server checks all applicable RoleBindings/ClusterRoleBindings│  │
│  │                                                                  │   │
│  │  Subject = ServiceAccount "myapp-sa" in namespace "production"   │   │
│  │      │                                                           │   │
│  │      ▼                                                           │   │
│  │  RoleBinding "myapp-binding" in namespace "production"?          │   │
│  │      └─► Role "pod-reader"                                       │   │
│  │              rules:                                               │   │
│  │              - apiGroups: [""]                                    │   │
│  │                resources: ["pods"]                                │   │
│  │                verbs: ["get","list","watch"]  ← MATCH!           │   │
│  │      │                                                           │   │
│  │      └─► ALLOWED ✓                                               │   │
│  │                                                                  │   │
│  │  If no match found → DENIED (default deny)                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Scope hierarchy:                                                        │
│  ClusterRole + ClusterRoleBinding  → cluster-wide access                 │
│  ClusterRole + RoleBinding         → namespace-scoped access             │
│  Role + RoleBinding                → namespace-scoped access             │
│  Role + ClusterRoleBinding         → NOT VALID                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Rolling Update — State Machine

```
TRIGGER: kubectl set image deployment/myapp app=myapp:v2

Initial state: ReplicaSet-v1 (3 pods running)

STEP 1: Create new ReplicaSet-v2
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 3 pods [●v1][●v1][●v1]    RS-v2: 0 pods            │
│  Service: → v1, v1, v1                                      │
└─────────────────────────────────────────────────────────────┘

STEP 2: Scale up RS-v2 by 1 (maxSurge=1)
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 3 pods [●v1][●v1][●v1]    RS-v2: 1 pod [◐v2]      │
│  Service: → v1, v1, v1  (v2 pod not ready yet)             │
└─────────────────────────────────────────────────────────────┘

STEP 3: v2 pod passes readiness probe → join Service
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 3 pods [●v1][●v1][●v1]    RS-v2: 1 pod [●v2]      │
│  Service: → v1, v1, v1, v2                                  │
└─────────────────────────────────────────────────────────────┘

STEP 4: Scale down RS-v1 by 1 (maxUnavailable=0 — no loss)
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 2 pods [●v1][●v1]          RS-v2: 1 pod [●v2]      │
│  Service: → v1, v1, v2                                      │
└─────────────────────────────────────────────────────────────┘

STEP 5-6: Repeat (scale up v2, then scale down v1)
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 1 pod  [●v1]               RS-v2: 3 pods [●v2×3]   │
│  Service: → v1, v2, v2, v2                                  │
└─────────────────────────────────────────────────────────────┘

STEP 7: Final state
┌─────────────────────────────────────────────────────────────┐
│  RS-v1: 0 pods (kept for rollback!)  RS-v2: 3 pods [●v2×3] │
│  Service: → v2, v2, v2              ← ALL traffic on v2     │
└─────────────────────────────────────────────────────────────┘

If something goes wrong at ANY step:
  kubectl rollout undo deployment/myapp
  → RS-v2 scales down, RS-v1 scales back up (fast, no rebuild)
```
