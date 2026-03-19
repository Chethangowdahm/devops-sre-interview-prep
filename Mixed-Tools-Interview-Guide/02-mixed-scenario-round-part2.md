# Mixed Tools Interview Guide - Part 2: Cloud + IaC + Monitoring Deep Mix

> Cross-tool questions that test how well you connect AWS/GCP + Terraform + Prometheus + Datadog

---

## SECTION 1: AWS + TERRAFORM COMBINED QUESTIONS

---

### Q1. [INTERVIEWER]: "You need to provision a production-ready EKS cluster on AWS. Walk me through the Terraform configuration."

**[TOOLS TESTED: Terraform + AWS + Kubernetes]**

**YOUR ANSWER:**
"Production-ready EKS needs several components. Here is my modular Terraform structure:

```hcl
# modules/eks/main.tf

# 1. VPC with public/private subnets
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "vendasta-prod-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false  # One per AZ for HA
  enable_dns_hostnames = true

  # Required tags for EKS to discover subnets
  private_subnet_tags = {
    "kubernetes.io/cluster/prod-cluster" = "shared"
    "kubernetes.io/role/internal-elb"    = "1"
  }
  public_subnet_tags = {
    "kubernetes.io/cluster/prod-cluster" = "shared"
    "kubernetes.io/role/elb"             = "1"
  }
}

# 2. EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.0.0"

  cluster_name    = "prod-cluster"
  cluster_version = "1.28"

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.private_subnets

  # IRSA (IAM Roles for Service Accounts) - no static credentials in pods
  enable_irsa = true

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.xlarge"]
      min_size       = 2
      max_size       = 20
      desired_size   = 3

      iam_role_additional_policies = {
        AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
      }
    }
  }
}
```

After the cluster, I provision: Cluster Autoscaler (using IRSA), AWS Load Balancer Controller, External DNS, cert-manager for TLS, and Prometheus Operator for monitoring.

**Key security considerations:**
- Worker nodes in private subnets only - never expose nodes directly to internet
- IRSA instead of node-level IAM - pods get only the permissions they need
- Encryption at rest for etcd using AWS KMS
- Private endpoint for cluster API server

**At Altruist:** I built this exact setup. The VPC subnet tags are critical - without them, EKS cannot auto-create load balancers. I learned this the hard way when our first ALB creation failed with a mysterious 'subnets not found' error."

---

### Q2. [INTERVIEWER]: "What is IRSA in AWS EKS and why is it important?"

**[TOOLS TESTED: AWS IAM + Kubernetes + Security]**

**YOUR ANSWER:**
"IRSA stands for IAM Roles for Service Accounts. It allows Kubernetes pods to assume AWS IAM roles without needing static credentials (access keys).

**How it works:**
1. EKS creates an OIDC identity provider for the cluster
2. You create an IAM role with a trust policy that allows the K8s service account to assume it
3. The pod uses the service account, which has the IAM role annotation
4. AWS SDK in the pod automatically gets temporary credentials via STS

```hcl
# IAM Role for a service that needs S3 access
resource "aws_iam_role" "s3_reader" {
  name = "payment-api-s3-reader"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = module.eks.oidc_provider_arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${module.eks.oidc_provider}:sub" = "system:serviceaccount:production:payment-api"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "s3_reader" {
  role       = aws_iam_role.s3_reader.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
```

```yaml
# Kubernetes ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/payment-api-s3-reader
```

**Why it matters:**
- No long-lived credentials stored anywhere (no access keys in Secrets or env vars)
- Credentials auto-rotate every 12 hours via STS
- Least-privilege: each pod gets only what IT needs, not what the node has
- Full audit trail in CloudTrail - every AWS API call is logged with the pod's identity

**At Altruist:** We started with EC2 instance profiles - every pod on the node had the same permissions. After a compromised pod incident, I migrated all services to IRSA. Each service now has its own minimal IAM role."

---

### Q3. [INTERVIEWER]: "How do you manage Terraform state in a team environment?"

**[TOOLS TESTED: Terraform + AWS S3/GCP GCS + Team workflows]**

**YOUR ANSWER:**
"Never use local state in a team. Remote state with locking is mandatory.

**Setup with GCS (what we use at Vendasta):**
```hcl
terraform {
  backend "gcs" {
    bucket  = "vendasta-terraform-state"
    prefix  = "prod/gke-cluster"
  }
}
```

**Setup with S3 + DynamoDB (what we used at Altruist):**
```hcl
terraform {
  backend "s3" {
    bucket         = "altruist-terraform-state"
    key            = "prod/eks/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"  # Prevents concurrent applies
  }
}
```

**State isolation strategy:**
- Split state by component: networking, cluster, applications - separate state files
- Never put everything in one state file - a corrupted state for networking would block application changes
- Use Terraform workspaces or directory structure for environment separation

**State locking:** DynamoDB/GCS provides distributed locking. If two engineers run `terraform apply` simultaneously, the second one waits or fails - preventing race conditions and state corruption.

**State security:**
- State files contain sensitive data (passwords, keys) in plaintext
- S3 bucket: block all public access, server-side encryption, versioning enabled, access logging
- GCS bucket: uniform bucket-level access, encryption, audit logging
- Never commit state files to Git - add `.terraform/` and `*.tfstate` to .gitignore

**At Vendasta:** We use GCS with a dedicated service account for Terraform that has minimal permissions. State access is logged in Cloud Audit Logs. We had one incident where two engineers ran apply simultaneously on overlapping resources - after that, we enforced CI-only applies with serialized jobs."

---

## SECTION 2: MONITORING + INCIDENT RESPONSE CROSS-TOOL QUESTIONS

---

### Q4. [INTERVIEWER]: "You get a Datadog alert at 2 AM: P99 latency for your checkout service spiked from 200ms to 3 seconds. Walk me through exactly what you do."

**[TOOLS TESTED: Datadog + Kubernetes + ArgoCD + Docker + Databases]**

**YOUR ANSWER:**
"This is a realistic P1 scenario. Here is my exact playbook:

**Minute 0-2: Acknowledge and assess**
- Acknowledge the PagerDuty alert (stops escalation)
- Open Datadog: check the latency graph - is it still climbing or has it stabilized?
- Check error rate alongside latency - are users getting errors or just slow responses?
- Check if this is all regions or one zone

**Minute 2-5: Correlation**
- Check Datadog deployment events overlay - was there a deploy in the last 30 minutes?
- If yes: go to ArgoCD, find the deploy time, confirm it matches the spike
- Check if any other services also degraded at the same time (cascade indicator)

**Minute 5-10: Dig into the trace**
- Open Datadog APM for checkout service
- Filter traces to p99 (slowest) - what does the flame graph show?
- Is the time in: application code? DB query? Redis? External API? Network?

**Scenario A: DB query is the bottleneck (most common)**
```sql
-- In Datadog DBM or directly on DB:
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Found: SELECT * FROM orders WHERE user_id = ? AND created_at > ? -- no index!
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';
-- Shows: Seq Scan (full table scan)

CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at);
-- CONCURRENTLY means no table lock - safe for production
```

**Scenario B: Recent deployment is the cause**
- Go to ArgoCD, click the checkout service app
- Click History - find the previous version
- Click Rollback - confirm
- ArgoCD rolls back the deployment in ~2 minutes
- Verify in Datadog: latency returns to 200ms baseline

**Scenario C: Pod is overwhelmed (not enough replicas)**
```bash
kubectl top pods -n production | grep checkout
# CPU 490m/500m - nearly at limit!
kubectl get hpa -n production
# CURRENT: 10/10 - HPA already at max!
# Temporarily increase maxReplicas
kubectl patch hpa checkout-api -n production -p '{"spec":{"maxReplicas":20}}'
```

**Minute 10-15: Resolution and communication**
- Post in #incidents: 'Root cause identified: [X]. Fix applied: [Y]. Monitoring for 10 min.'
- Watch Datadog latency graph return to baseline
- After stable for 10 min: 'P1 resolved. Post-mortem to follow.'

**At Vendasta:** I handled this exact scenario. A checkout latency spike at 11 PM was caused by a missing composite index on `(user_id, created_at)`. The table had grown to 50M rows and the query was doing a full table scan. Adding the index with CONCURRENTLY resolved it without downtime. Post-mortem led to mandatory EXPLAIN ANALYZE reviews in our DB migration CI check."

---

### Q5. [INTERVIEWER]: "How do you implement multi-environment infrastructure with Terraform? Dev, staging, production."

**[TOOLS TESTED: Terraform + GCP/AWS]**

**YOUR ANSWER:**
"Two main approaches - I'll explain both and which I prefer:

**Approach 1: Directory-based environments (my preferred)**
```
infra/
  modules/
    gke/          # Reusable GKE module
    networking/   # Reusable VPC module
  environments/
    dev/
      main.tf     # Calls modules with dev values
      terraform.tfvars
    staging/
      main.tf
      terraform.tfvars
    production/
      main.tf
      terraform.tfvars
```

Each environment has its own state file. `environments/production/main.tf` calls the same modules as dev but with production-sized values:
```hcl
# environments/production/main.tf
module "gke" {
  source = "../../modules/gke"

  environment     = "production"
  min_nodes       = 3
  max_nodes       = 30
  machine_type    = "n2-standard-8"  # Bigger than dev
  enable_shielded = true              # Production security
}
```

**Approach 2: Terraform Workspaces**
```bash
terraform workspace new production
terraform workspace select production
terraform apply -var-file=production.tfvars
```
Workspaces share code but use different state. Less separation than directories - I prefer directories for clearer blast radius isolation.

**CI/CD integration:**
- Dev: auto-apply on every merge to main
- Staging: auto-apply with approval step
- Production: manual trigger only, require two approvers in the PR

**At Vendasta:** We use directory-based with a Makefile wrapper. `make plan ENV=production` runs the right plan. `make apply ENV=production` requires a second person to approve the PR before GitHub Actions runs it."

---

### Q6. [INTERVIEWER]: "Your Prometheus is showing high cardinality warnings. What is cardinality and how do you fix it?"

**[TOOLS TESTED: Prometheus + Monitoring best practices]**

**YOUR ANSWER:**
"Cardinality is the number of unique time series in Prometheus. It is calculated as the product of all label value combinations.

**Example of high cardinality (bad):**
```go
// BAD: user_id has millions of unique values!
var requestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total"},
    []string{"method", "path", "status", "user_id"},  // user_id explodes cardinality
)
```
10 methods x 100 paths x 5 statuses x 1,000,000 users = 5 BILLION time series. Prometheus runs out of memory.

**Fix: Never use high-cardinality values as labels**
```go
// GOOD: only bounded label values
var requestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total"},
    []string{"method", "path", "status"},  // All bounded: 10 x 100 x 5 = 5000 series
)
// Log user_id in structured logs instead
```

**Rules for Prometheus labels:**
- Labels must have a BOUNDED set of values (finite number of possible values)
- Good labels: method, status_code, service, environment, region
- Bad labels: user_id, request_id, IP address, email, UUID

**Diagnosing high cardinality:**
```promql
# Find the metrics with most time series
topk(10, count by(__name__)({__name__=~".+"}))

# Find which label values are exploding
count by(path)(http_requests_total) | sort_desc
```

**At Vendasta:** We had a developer add `request_id` as a Prometheus label. Prometheus memory went from 2GB to 18GB in 6 hours. I found the metric using the topk query above, removed the bad label via a new deployment, and Prometheus recovered within 30 minutes after the scrape data expired."

---

### Q7. [INTERVIEWER]: "How do you handle secrets in a Kubernetes + GCP/AWS environment without exposing them in Git or environment variables?"

**[TOOLS TESTED: Kubernetes + GCP Secret Manager + AWS Secrets Manager + Security]**

**YOUR ANSWER:**
"Secret management has evolved significantly. Here is the production-grade approach I use:

**On GCP (at Vendasta):**
1. Secrets stored in GCP Secret Manager - never in Git, never in K8s Secrets as plaintext
2. The application pod uses Workload Identity - no static credentials
3. At runtime, the app fetches secrets from Secret Manager via API

```go
// In application code - fetch at startup
func getSecret(ctx context.Context, name string) (string, error) {
    client, _ := secretmanager.NewClient(ctx)
    result, err := client.AccessSecretVersion(ctx, &secretmanagerpb.AccessSecretVersionRequest{
        Name: fmt.Sprintf("projects/vendasta-prod/secrets/%s/versions/latest", name),
    })
    return string(result.Payload.Data), err
}
```

**Alternative: External Secrets Operator (ESO)**
ESO is a Kubernetes operator that syncs secrets from external stores (Vault, AWS Secrets Manager, GCP Secret Manager) into K8s Secrets automatically.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-api-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: payment-api-env
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: payment-api-db-password
  - secretKey: STRIPE_KEY
    remoteRef:
      key: payment-api-stripe-key
```

ESO creates a K8s Secret that is auto-rotated when the source in Secret Manager changes. The pod just reads from the K8s Secret as normal - no code changes needed.

**Rotation:** Secret Manager supports automatic rotation with Cloud Functions. When a DB password rotates, ESO syncs it to the K8s Secret within 1 hour, and the app hot-reloads without restart (for connection-pool based apps) or does a rolling restart.

**At Vendasta:** We use ESO with GCP Secret Manager for all application secrets. Zero secrets in Git, zero plaintext in etcd, automatic rotation every 90 days. Passed SOC2 audit for secret management."

---

### Q8. [INTERVIEWER]: "How do you handle Nginx configuration in a Kubernetes environment? When would you use Nginx vs cloud-native load balancers?"

**[TOOLS TESTED: Nginx + Kubernetes Ingress + AWS ALB + GCP LB]**

**YOUR ANSWER:**
"In Kubernetes, Nginx is typically deployed as the Ingress Controller - it handles all HTTP/HTTPS routing within the cluster.

**How it works:**
1. Nginx Ingress Controller runs as a Deployment in the cluster
2. A cloud Load Balancer (GCP HTTPS LB or AWS ALB) sits in front of Nginx
3. Kubernetes Ingress resources define routing rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.vendasta.com
    secretName: api-tls
  rules:
  - host: api.vendasta.com
    http:
      paths:
      - path: /v1/payment
        pathType: Prefix
        backend:
          service:
            name: payment-api
            port:
              number: 80
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-api
            port:
              number: 80
```

**Nginx vs cloud-native load balancers:**

**Use Nginx Ingress when:**
- You need advanced routing: header-based routing, canary %, regex paths
- You need rate limiting, IP allowlisting, custom error pages
- You want consistent behavior across clouds (portability)
- You need sticky sessions, upstream keepalive tuning

**Use Cloud-native (AWS ALB, GCP HTTPs LB) when:**
- You want native WAF integration (AWS WAF, Cloud Armor)
- You need global load balancing across regions
- You need minimal operational overhead (fully managed)
- Cost: cloud LBs can be cheaper at scale than running Nginx pods

**At Vendasta:** We use Nginx Ingress for cluster-level routing and GCP Global Load Balancer in front of it for global anycast, SSL termination, and Cloud Armor WAF. Best of both worlds."

---

*Next: File 03 - Advanced Troubleshooting Scenarios (Pure Production Incident Cases)*
