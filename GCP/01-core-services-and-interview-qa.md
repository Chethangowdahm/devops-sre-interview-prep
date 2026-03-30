# GCP - Core Services & Interview Q&A
> Senior DevOps/SRE Level | Vendasta Production Experience

---

## CORE GCP SERVICES

### Compute
| Service | AWS Equivalent | Use Case |
|---|---|---|
| **Compute Engine (GCE)** | EC2 | Virtual machines |
| **GKE** | EKS | Managed Kubernetes |
| **Cloud Run** | ECS Fargate / App Runner | Serverless containers |
| **Cloud Functions** | Lambda | Serverless functions |
| **App Engine** | Elastic Beanstalk | PaaS |

### Storage
| Service | AWS Equivalent | Use Case |
|---|---|---|
| **Cloud Storage (GCS)** | S3 | Object storage |
| **Persistent Disk** | EBS | Block storage for VMs |
| **Filestore** | EFS | NFS file storage |
| **Cloud SQL** | RDS | Managed MySQL/PostgreSQL |
| **Cloud Spanner** | Aurora | Globally distributed SQL |
| **Firestore** | DynamoDB | NoSQL document DB |
| **BigQuery** | Redshift | Data warehouse/analytics |
| **Cloud Bigtable** | DynamoDB (wide-column) | Time series, analytics |

### Networking
| Service | AWS Equivalent | Use Case |
|---|---|---|
| **VPC** | VPC | Virtual private network |
| **Cloud Load Balancing** | ELB | Global HTTP(S)/TCP/UDP LB |
| **Cloud CDN** | CloudFront | Edge caching |
| **Cloud DNS** | Route 53 | DNS management |
| **Cloud NAT** | NAT Gateway | Outbound internet for private IPs |

### Security & Identity
| Service | AWS Equivalent | Use Case |
|---|---|---|
| **IAM** | IAM | Identity and access management |
| **Workload Identity** | IRSA | K8s pods → GCP service accounts |
| **Secret Manager** | Secrets Manager | Secret storage/rotation |
| **KMS** | KMS | Encryption key management |
| **Cloud Armor** | WAF + Shield | DDoS + WAF |
| **VPC Service Controls** | AWS PrivateLink (partial) | API perimeter security |

### Monitoring & Ops
| Service | AWS Equivalent | Use Case |
|---|---|---|
| **Cloud Monitoring** | CloudWatch Metrics | Metrics + dashboards |
| **Cloud Logging** | CloudWatch Logs | Log management |
| **Cloud Trace** | X-Ray | Distributed tracing |
| **Cloud Profiler** | - | CPU/heap profiling in production |
| **Error Reporting** | - | Aggregated error reporting |

---

## GKE DEEP DIVE

### GKE Cluster Types
- **Standard**: You manage node pools, OS, kubelet config
- **Autopilot**: Google manages nodes, scheduling, scaling. Pay per pod resource.

### GKE Autopilot (Production at Vendasta)
- No node management needed
- Automatic node provisioning and scaling
- Pod-level billing (more cost-efficient for variable workloads)
- Security hardened by default

### Workload Identity (GKE equivalent of IRSA)
```yaml
# 1. K8s ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: my-app@myproject.iam.gserviceaccount.com
```
```bash
# 2. Bind K8s SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  my-app@myproject.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member serviceAccount:myproject.svc.id.goog[production/my-app-sa]
```

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: GCP VPC vs AWS VPC differences?
**GCP VPC is global** — a single VPC spans all regions. Subnets are regional.
**AWS VPC is regional** — one VPC per region, need VPC peering for cross-region.

This makes GCP networking simpler for multi-region deployments.

### Q2: What is Cloud Run and when would you use it over GKE?
Cloud Run runs stateless containers serverlessly. Auto-scales to zero.

| Aspect | Cloud Run | GKE |
|---|---|---|
| Management | Fully managed | Control plane managed |
| Scaling | Auto (0 to N) | HPA (min 1 per default) |
| Stateful | No | Yes (StatefulSets) |
| Long-running | Limited (24h timeout) | Yes |
| Cost | Per request | Per node/pod resource |
| Use case | APIs, webhooks, event handlers | Complex microservices, full apps |

### Q3: How does Workload Identity work in GKE?
Workload Identity allows GKE pods to impersonate GCP Service Accounts without storing credentials.

**Flow**:
1. Pod has a K8s ServiceAccount
2. K8s SA is annotated with GCP SA
3. GCP SA IAM policy allows K8s SA to impersonate it
4. When pod calls GCP APIs, it automatically gets a token for the GCP SA

**No keys stored anywhere** — credentials are short-lived tokens from GCP metadata server.

### Q4: What is GCS and how does lifecycle management work?
GCS is Google Cloud Storage (equivalent to S3). Object storage with global CDN integration.

**Storage classes**:
- Standard: Hot data, frequently accessed
- Nearline: Once/month access. Min 30-day storage.
- Coldline: Once/quarter. Min 90-day.
- Archive: Once/year. Min 365-day.

**Lifecycle example** (via Terraform):
```hcl
resource "google_storage_bucket" "logs" {
  lifecycle_rule {
    condition { age = 30 }
    action { type = "SetStorageClass"; storage_class = "NEARLINE" }
  }
  lifecycle_rule {
    condition { age = 365 }
    action { type = "Delete" }
  }
}
```

### Q5: How does Cloud Load Balancing differ from AWS?
GCP's HTTP(S) LB is **global** — single anycast IP, traffic routed to nearest healthy backend.
AWS ALB is regional — need Route 53 or Global Accelerator for global distribution.

GCP LB types:
- **HTTP(S) LB**: Global, L7, supports Cloud CDN, Cloud Armor
- **TCP/UDP LB**: Regional, L4
- **Internal HTTP(S) LB**: Private, for internal microservices
- **Network LB**: External, L4, ultra-low latency

### Q6: Explain Cloud Build vs Jenkins
Cloud Build is GCP's managed CI service.

| Aspect | Cloud Build | Jenkins |
|---|---|---|
| Management | Fully managed | Self-hosted |
| Integration | Native GCP (GCR, GKE, GCS) | Plugin-based |
| Cost | Pay per build minute | EC2/VM costs |
| Config | cloudbuild.yaml | Jenkinsfile |
| Scaling | Auto | Configure agents |

```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA', '.']
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA']
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  args: ['gke-deploy', 'run', '--filename=k8s/', '--cluster=production']
images:
- 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA'
```

### Q7: How do you manage GCP infrastructure at scale?
1. **Terraform**: For all GCP resources (GKE, Cloud SQL, VPC, IAM)
2. **Config Connector**: Manage GCP resources via K8s CRDs
3. **Google Cloud Deploy**: Managed CD for GKE (alternative to ArgoCD)
4. **Organization Policies**: Enforce constraints across all projects
5. **Resource Manager**: Organize projects into folders, apply IAM at org level
6. **Shared VPC**: Central networking, managed by platform team, shared across projects

---

## REAL-WORLD: GKE Production Setup (Vendasta-style)

```hcl
# Terraform - GKE Autopilot Cluster
resource "google_container_cluster" "production" {
  name     = "production-cluster"
  location = "us-central1"

  enable_autopilot = true

  networking_config {
    enable_multi_networking = true
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  addons_config {
    horizontal_pod_autoscaling { disabled = false }
    http_load_balancing { disabled = false }
    gce_persistent_disk_csi_driver_config { enabled = true }
  }

  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "APISERVER"]
  }
}
```
