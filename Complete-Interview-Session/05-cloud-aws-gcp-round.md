# Complete Interview Session — File 5: Cloud AWS & GCP Round

> **Round Type:** Cloud architecture deep dive — expect multi-service questions and design tradeoffs
> **Your Edge:** 3 years on AWS at Altruist, 2+ years on GCP/GKE at Vendasta. You know both.

---

## SECTION 1: AWS DEEP DIVE

---

### Q1. Explain AWS VPC design. How did you design it for production?

**[YOUR ANSWER]:**

"A VPC is an isolated virtual network in AWS. In production, I follow this multi-tier architecture:

```
VPC: 10.0.0.0/16

  Public Subnets (10.0.1.0/24, 10.0.2.0/24) — Multi-AZ
  Hosts: Application Load Balancer, NAT Gateway, Bastion Host

  Private Subnets (10.0.10.0/24, 10.0.11.0/24) — Multi-AZ
  Hosts: EC2 application servers, EKS worker nodes

  Database Subnets (10.0.20.0/24, 10.0.21.0/24) — Multi-AZ
  Hosts: RDS, ElastiCache — NO internet access
```

**Security layers:**
- Internet Gateway: Only for public subnet
- NAT Gateway: Lets private instances reach internet for updates, not accessible from internet
- Security Groups: Instance-level firewall — stateful (return traffic automatically allowed)
- NACLs: Subnet-level firewall — stateless (explicitly allow return traffic)
- VPC Flow Logs: For audit and network troubleshooting

**At Altruist:** I designed this exact architecture. RDS was only accessible from application subnet security group — port 5432 from app-sg only. Zero direct internet access to database tier."

---

### Q2. What is IAM in AWS? Explain roles vs users vs policies.

**[YOUR ANSWER]:**

"AWS IAM is the identity and access management service — controls WHO can do WHAT on which resource.

**IAM Users:** Long-lived identities with permanent credentials (access key + secret). Used for humans or legacy apps. Avoid for applications in EC2/EKS.

**IAM Roles:** Temporary credentials assumed by services, EC2 instances, Lambda, or EKS pods. No long-lived keys. Best practice for all applications.

**IAM Policies:** JSON documents defining permissions. Attached to users, groups, or roles.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::vendasta-data/*"
  }]
}
```

**IRSA (IAM Roles for Service Accounts):** Kubernetes pods on EKS can assume IAM roles via service account annotation — no credentials stored in pods.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/payment-api-role
```

**At Altruist:** I replaced all IAM user credentials in EC2 instances with instance profiles + roles. Eliminated the entire class of 'leaked credentials' security incidents."

---

### Q3. Explain AWS EKS vs self-managed Kubernetes. What are the tradeoffs?

**[YOUR ANSWER]:**

"**EKS (Elastic Kubernetes Service):**
- AWS manages the control plane — kube-apiserver, etcd, scheduler
- Highly available control plane by default across 3 AZs
- Integrated with AWS services: IAM, ALB Ingress, EBS CSI, CloudWatch
- You manage worker nodes via Node Groups or Fargate
- Cost: Control plane ~$0.10/hr per cluster

**Self-managed K8s:**
- You own everything — control plane HA, etcd backups, upgrades
- Full flexibility — any version, any configuration
- High operational overhead — upgrade path is complex
- Use case: very specific compliance or cost requirements

**Recommendation:** EKS for 99% of production workloads. The control plane cost is trivial compared to the engineering hours saved.

**At Altruist:** We ran self-managed K8s on EC2 initially — upgrading from 1.20 to 1.24 took 3 days and had a 2-hour downtime. After that, I proposed and migrated to EKS, and subsequent upgrades took 30 minutes with zero downtime."

---

### Q4. How do you optimize AWS costs? Give specific examples.

**[YOUR ANSWER]:**

"Cost optimization has multiple levers:

**Compute:**
- Reserved Instances or Savings Plans for predictable workloads (up to 72% savings)
- Spot Instances for fault-tolerant workloads — dev/test, batch, non-critical
- Right-sizing: Check CPU utilization — most EC2s run at <20% CPU. Use AWS Compute Optimizer.

**Storage:**
- S3 lifecycle policies: Move to Infrequent Access after 30 days, Glacier after 90 days
- Delete unattached EBS volumes (common after instance terminations)
- Enable S3 Intelligent-Tiering for unpredictable access patterns

**Networking:**
- Avoid cross-AZ data transfer where possible (put app and DB in same AZ for read replicas)
- Use VPC endpoints for S3/DynamoDB — avoid NAT gateway charges

**At Altruist:** I identified that our dev environment ran 24/7. I wrote a Lambda function to stop EC2 instances outside business hours. Monthly savings: $800 just on dev compute. Also moved old log archives to S3 Glacier — saved $200/month in storage."

---

### Q5. Explain S3 security best practices.

**[YOUR ANSWER]:**

"S3 is highly capable but also a common source of data breaches. Best practices:

1. **Block Public Access:** Enable 'Block All Public Access' at account level unless you specifically need public buckets
2. **Bucket Policies:** Use resource-based policies to enforce HTTPS only and deny from specific IPs
3. **Server-Side Encryption:** Enable SSE-S3 or SSE-KMS by default
4. **Versioning:** Enable for critical data — allows recovery from accidental deletes
5. **Access Logging:** Enable S3 server access logs for audit trails
6. **VPC Endpoints:** Access S3 from within VPC without going through internet

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
  }]
}
```

**At Altruist:** Found an S3 bucket with public read access that was storing internal reports. Immediately blocked public access and moved access behind IAM roles."

---

## SECTION 2: GCP DEEP DIVE

---

### Q6. Explain GKE vs EKS. What makes GKE better for some use cases?

**[YOUR ANSWER]:**

"Both are managed Kubernetes services but with different strengths:

**GKE advantages:**
- Google invented Kubernetes — deepest integration and latest features first
- GKE Autopilot: Fully managed worker nodes — no node pool management
- Workload Identity: Cleaner than IRSA for pod-level IAM (no webhook needed)
- Release channels (rapid/regular/stable) for managed upgrades
- Binary Authorization: Enforces only signed images can be deployed

**EKS advantages:**
- Better integration with AWS ecosystem (RDS, SQS, SNS, etc.)
- Fargate for serverless nodes
- Larger AWS customer base — more community examples

**At Vendasta (GKE):** We use GKE Standard with 3 node pools — general, high-memory, and spot instances for batch. Cluster upgrades are managed via release channel — zero-touch."

---

### Q7. What is GCP Workload Identity and how does it work?

**[YOUR ANSWER]:**

"Workload Identity allows GKE pods to authenticate as GCP service accounts without any downloaded key files.

**How it works:**
1. Create a GCP Service Account (GSA)
2. Create a Kubernetes Service Account (KSA)
3. Bind KSA to GSA with IAM policy
4. Annotate KSA with GSA email
5. Pod using KSA automatically gets GSA credentials via metadata server

```bash
# Create binding
gcloud iam service-accounts add-iam-policy-binding \
  payment-api@vendasta-prod.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member 'serviceAccount:vendasta-prod.svc.id.goog[production/payment-api-ksa]'

# Annotate Kubernetes SA
kubectl annotate serviceaccount payment-api-ksa \
  iam.gke.io/gcp-service-account=payment-api@vendasta-prod.iam.gserviceaccount.com
```

**At Vendasta:** All microservices use Workload Identity. No service account keys exist anywhere — zero risk of leaked credentials."

---

### Q8. Explain GCP networking — VPC, subnets, Cloud NAT, VPC peering.

**[YOUR ANSWER]:**

"GCP networking has some key differences from AWS:

**GCP VPC:** Global — one VPC spans all regions. Subnets are regional (not AZ-specific like AWS).

**Subnets:** Each subnet has a primary IP range + optional secondary ranges (used by GKE Pods and Services via VPC-native clusters).

**Cloud NAT:** Managed NAT service for private instances to reach internet without public IPs. No single point of failure.

**VPC Peering:** Connects two VPCs privately. Non-transitive — if A peers with B and B peers with C, A cannot reach C.

**Shared VPC:** Allows multiple projects to share a single host VPC — centralized networking control with project isolation.

**At Vendasta:** We use VPC-native GKE clusters with secondary IP ranges. GKE Pods get IPs from a dedicated Pod IP range, visible in GCP routing tables — efficient traffic without overlay network overhead."

---

### Q9. How do you manage multi-environment infrastructure on GCP with Terraform?

**[YOUR ANSWER]:**

"I use a workspace-per-environment pattern with shared modules:

```
infrastructure/
  modules/
    gke-cluster/     # Reusable GKE module
    cloud-sql/       # Reusable CloudSQL module
    vpc/             # Reusable VPC module
  environments/
    dev/
      main.tf        # Calls modules with dev-specific variables
      terraform.tfvars
    staging/
      main.tf
      terraform.tfvars
    production/
      main.tf
      terraform.tfvars  # Larger node counts, multi-AZ
```

**State management:**
- Remote state in GCS bucket per environment
- State locking via GCS object versioning
- Never mix dev/prod state files

```hcl
terraform {
  backend "gcs" {
    bucket = "vendasta-terraform-state"
    prefix = "production/gke-cluster"
  }
}
```

**At Vendasta:** Promoted infrastructure changes from dev to staging to production using the same module, just different variable files. When a GKE cluster config works in staging for 48h, same Terraform gets applied to production."

---

### Q10. Scenario: You notice your GCP costs spiked 40% last month. How do you investigate?

**[YOUR ANSWER]:**

"I'd approach this systematically:

**Step 1 — GCP Cost Explorer / BigQuery billing export:**
- Break down by service, project, label
- Which service grew the most? Compute? Networking? Storage?

**Step 2 — Common culprits:**
- Compute: Uncleaned node pools, VMs left running after testing
- Networking: Unexpected data egress (especially to internet or cross-region)
- Storage: GCS bucket growing unexpectedly, missing lifecycle policies
- Load Balancers: Unused forwarding rules still billing

**Step 3 — Label analysis:**
- All resources should have team, service, and environment labels
- Untagged resources often hide costs

**Step 4 — Set budget alerts:**
```bash
gcloud billing budgets create --billing-account=ACCOUNT_ID \
  --display-name='Monthly Budget Alert' \
  --budget-amount=10000USD \
  --threshold-rule=percent=0.8,basis=CURRENT_SPEND
```

**Real example at Vendasta:** Cost spike traced to egress from GKE pods calling an external API in a loop — a bug where a retry loop had no backoff. Fixed the bug, added rate limiting, cost returned to normal. Also added egress metric dashboards to catch this class early."

---

*Next: Move to File 06 — Terraform & Ansible Round*
