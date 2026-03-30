# GKE — Google Kubernetes Engine (6 YOE)

---

## What is GKE?

GKE is Google's **managed Kubernetes service**. Google manages the control plane (API server, etcd, scheduler, controller manager) — you only manage worker nodes and workloads.

```
Self-managed K8s:  YOU manage everything (control plane + nodes)
GKE Standard:      Google manages control plane. You manage node pools.
GKE Autopilot:     Google manages control plane + nodes. You only manage pods.
```

---

## GKE Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      GCP PROJECT                                     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   GKE CLUSTER                                │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │  CONTROL PLANE (Google Managed)                     │    │   │
│  │  │  API Server │ etcd │ Scheduler │ Controllers        │    │   │
│  │  │  Cloud Controller Manager (GCP integrations)        │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │                          │                                   │   │
│  │  ┌───────────────────────┼─────────────────────────────┐    │   │
│  │  │  NODE POOLS           │                             │    │   │
│  │  │                       │                             │    │   │
│  │  │  ┌─────────────┐  ┌───┴─────────┐  ┌────────────┐  │    │   │
│  │  │  │ Default Pool│  │  Spot Pool  │  │  GPU Pool  │  │    │   │
│  │  │  │ n2-standard │  │  n2-spot    │  │  a100      │  │    │   │
│  │  │  │ 3 nodes     │  │  5 nodes    │  │  2 nodes   │  │    │   │
│  │  │  └─────────────┘  └─────────────┘  └────────────┘  │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │                                                              │   │
│  │  Integrated GCP Services:                                    │   │
│  │  GCR/Artifact Registry  Cloud Load Balancing  Cloud Armor   │   │
│  │  Cloud SQL (via proxy)  GCS                   Secret Mgr    │   │
│  │  Cloud Logging          Cloud Monitoring       IAM/Workload  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GKE vs Self-Managed K8s

| Feature | Self-managed | GKE Standard | GKE Autopilot |
|---------|-------------|--------------|---------------|
| Control plane management | You | Google | Google |
| Node management | You | You | Google |
| Auto-upgrade | Manual | Configurable | Automatic |
| Pricing | Infra cost | Infra + mgmt fee | Per pod resource |
| Flexibility | Full | Full | Pod-level only |
| Best for | On-prem/custom | Most prod use cases | Hands-off prod |

---

## Creating a GKE Cluster

```bash
# Standard cluster with autoscaling node pool
gcloud container clusters create my-cluster \
  --project=my-project \
  --region=us-central1 \
  --release-channel=regular \
  --num-nodes=3 \
  --min-nodes=2 \
  --max-nodes=10 \
  --enable-autoscaling \
  --machine-type=n2-standard-4 \
  --disk-type=pd-ssd \
  --disk-size=100 \
  --enable-ip-alias \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-network-policy \
  --workload-pool=my-project.svc.id.goog \  # Workload Identity
  --enable-shielded-nodes \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM,WORKLOAD

# Get credentials
gcloud container clusters get-credentials my-cluster \
  --region=us-central1 \
  --project=my-project
```

---

## Node Pools

```bash
# Add a Spot node pool (cheaper, ~60-91% discount, but preemptible)
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --spot \
  --machine-type=n2-standard-4 \
  --num-nodes=0 \
  --min-nodes=0 \
  --max-nodes=20 \
  --enable-autoscaling \
  --node-taints=cloud.google.com/gke-spot=true:NoSchedule

# Tolerate spot nodes in deployment
spec:
  tolerations:
  - key: cloud.google.com/gke-spot
    operator: Equal
    value: "true"
    effect: NoSchedule
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: cloud.google.com/gke-spot
            operator: In
            values: ["true"]
```

---

## Workload Identity (Critical Security Concept)

Replace service account key files with GCP IAM-based pod authentication.

```
Without Workload Identity:
  Pod → needs JSON key file (secret) → GCP API
  Problem: Key rotation, key leakage

With Workload Identity:
  Pod (K8s ServiceAccount) → automatically maps to → GCP IAM ServiceAccount
  No key files needed!
```

```bash
# 1. Create GCP service account
gcloud iam service-accounts create myapp-sa \
  --project=my-project

# 2. Grant GCS access
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:myapp-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# 3. Allow K8s ServiceAccount to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  myapp-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[production/myapp-ksa]"

# 4. Annotate K8s ServiceAccount
kubectl annotate serviceaccount myapp-ksa \
  --namespace production \
  iam.gke.io/gcp-service-account=myapp-sa@my-project.iam.gserviceaccount.com
```

```yaml
# Pod uses the annotated KSA
spec:
  serviceAccountName: myapp-ksa
```

---

## GKE Ingress — Cloud Load Balancing Integration

```
Internet → Cloud Armor (WAF/DDoS) → GLB (HTTPS LB) → GKE Ingress → Service → Pods
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"                      # Use GCE LB
    kubernetes.io/ingress.global-static-ip-name: "myapp-ip" # Reserve static IP
    networking.gke.io/managed-certificates: "myapp-cert"    # Google-managed TLS
    networking.gke.io/v1beta1.FrontendConfig: "myapp-fec"   # HTTPS redirect
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
```

---

## GKE Autopilot

```bash
# Create Autopilot cluster
gcloud container clusters create-auto my-autopilot-cluster \
  --region=us-central1 \
  --project=my-project

# Key differences vs Standard:
# - No node pool management
# - Pods billed per resource request (not per node)
# - No SSH to nodes
# - No DaemonSets (unless approved by Google)
# - Minimum resource requests enforced
# - Automatically scales down to 0 when no pods
```

---

## GKE Upgrade Strategy

```
Release channels:
  Rapid   → Latest (for testing, gets new K8s quickly)
  Regular → Balanced (recommended for most prod, ~1 month delay)
  Stable  → Slowest, most tested (conservative prod)

Upgrade modes:
  Surge upgrade (default): Creates surge nodes → migrates pods → removes old
  Blue/Green node pool:    Create new pool → migrate → delete old

Best practice for zero-downtime upgrades:
  1. Set PodDisruptionBudget
  2. Use Surge upgrades with sufficient surge capacity
  3. Set terminationGracePeriodSeconds appropriately
  4. Test in staging first
```

---

## Private Cluster & VPC-native Networking

```
Private Cluster:
  - Master has no public IP
  - Nodes have no public IPs
  - Access master via Authorized Networks or Private Google Access
  - Workloads reach internet via Cloud NAT

VPC-native (alias IP) vs routes-based:
  VPC-native: Pod IPs are real IPs in VPC subnet (recommended)
  Routes-based: Legacy, uses routes table (limited scale)
```

---

## GKE Monitoring Integration

```bash
# Cloud Logging auto-integration
# Logs available in Cloud Logging automatically when --logging=WORKLOAD

# View in gcloud
gcloud logging read 'resource.type="k8s_container" resource.labels.cluster_name="my-cluster"' \
  --project=my-project \
  --limit=50

# Cloud Monitoring dashboards
# GKE workloads dashboard available by default
# Custom metrics via Prometheus → Cloud Monitoring adapter
```

---

## Cost Optimization Strategies

| Strategy | Savings |
|---------|---------|
| Spot/Preemptible node pools for batch | 60-91% |
| Cluster Autoscaler (scale to 0 for non-prod) | 40-80% |
| Committed use discounts (1-3 year) | 37-55% |
| Right-size with VPA recommendations | 20-40% |
| Use E2 machine type (cost-optimized) | ~30% vs N2 |
| GKE Autopilot for variable workloads | Pay per pod |

---

## Interview Questions — GKE (6 YOE Level)

**Q: What is Workload Identity and why is it preferred over service account keys?**
> Workload Identity maps K8s Service Accounts to GCP IAM Service Accounts. Pods automatically get GCP credentials without needing JSON key files. Eliminates key rotation burden, prevents key leakage, audited via Cloud Audit Logs. Best practice for GKE workloads accessing GCP APIs.

**Q: How do you upgrade a GKE cluster with zero downtime?**
> Use surge upgrades: set surgeUpgrade to create extra nodes before draining. Ensure PDBs are in place, readiness probes work, terminationGracePeriodSeconds is adequate. Use regional clusters (control plane HA across zones). Test upgrade in non-prod first. Monitor with Cloud Monitoring during the upgrade.

**Q: What's the difference between regional and zonal GKE clusters?**
> **Zonal**: Control plane in one zone. Single point of failure. Cheaper. **Regional**: Control plane replicated across 3 zones. Nodes distributed across zones. HA — survives zone failure. ~$0.10/hr management fee for both, but regional is strongly recommended for production.

**Q: How does GKE integrate with GCP LoadBalancing?**
> `type: LoadBalancer` Service → Cloud Controller Manager creates a Network Load Balancer (L4). `Ingress` with `gce` class → creates an HTTPS Load Balancer (L7) with global anycast IPs, managed SSL certs, Cloud CDN, Cloud Armor integration.

**Q: How do you restrict pod-to-pod communication in GKE?**
> Enable Network Policy (`--enable-network-policy`) at cluster creation (uses Calico or Dataplane V2). Define NetworkPolicy objects to whitelist allowed traffic. With GKE Dataplane V2 (eBPF-based), you also get network policy logging.
