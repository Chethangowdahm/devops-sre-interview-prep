# GCP Services for SRE/DevOps (6 YOE)

---

## Core GCP Services Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GCP SERVICES                                 │
│                                                                      │
│  COMPUTE              STORAGE               NETWORKING               │
│  ├─ GKE               ├─ GCS (object)        ├─ VPC                  │
│  ├─ GCE (VMs)         ├─ Cloud SQL           ├─ Cloud Load Balancing │
│  ├─ Cloud Run         ├─ Cloud Spanner        ├─ Cloud CDN            │
│  └─ Cloud Functions   ├─ Firestore/Datastore  ├─ Cloud Armor (WAF)   │
│                        ├─ Memorystore (Redis)  ├─ Cloud DNS            │
│  CI/CD                └─ Persistent Disk      └─ Cloud NAT            │
│  ├─ Cloud Build                                                       │
│  ├─ Artifact Registry  MESSAGING              SECURITY                │
│  └─ Cloud Deploy       ├─ Pub/Sub             ├─ IAM                  │
│                        ├─ Cloud Tasks         ├─ Secret Manager       │
│  OBSERVABILITY         └─ Cloud Scheduler     ├─ KMS                  │
│  ├─ Cloud Logging                             ├─ Binary Authorization │
│  ├─ Cloud Monitoring   DATA/ML                └─ Security Command Ctr │
│  ├─ Cloud Trace        ├─ BigQuery                                    │
│  └─ Error Reporting    ├─ Dataflow                                    │
│                        └─ Vertex AI                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Google Cloud Storage (GCS)

Object storage. Infinitely scalable. Used for backups, artifacts, static assets, data lakes.

```bash
# Create bucket
gsutil mb -p my-project -l us-central1 -b on gs://my-bucket

# Storage classes:
#   STANDARD    → frequently accessed (~$0.02/GB/month)
#   NEARLINE    → accessed < once/month (~$0.01/GB)
#   COLDLINE    → accessed < once/quarter
#   ARCHIVE     → accessed < once/year (cheapest, high retrieval cost)

# Lifecycle policy (auto-transition/delete)
cat > lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
EOF
gsutil lifecycle set lifecycle.json gs://my-bucket

# Operations
gsutil cp local-file.tar.gz gs://my-bucket/backups/
gsutil rsync -r ./build gs://my-bucket/static/
gsutil ls -la gs://my-bucket/
gsutil du -sh gs://my-bucket/

# Signed URLs (temporary access without credentials)
gsutil signurl -d 1h service-account-key.json gs://my-bucket/file.tar.gz
```

**Use cases for SRE**:
- Store Docker build artifacts, Terraform state, backup files
- Static website hosting
- Audit logs archival from Cloud Logging
- Data pipeline staging

---

## 2. Cloud SQL

Managed relational database (MySQL, PostgreSQL, SQL Server).

```bash
# Create PostgreSQL instance
gcloud sql instances create prod-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-4 \
  --region=us-central1 \
  --availability-type=REGIONAL \    # HA with automatic failover
  --backup-start-time=03:00 \
  --enable-bin-log \
  --retained-backups-count=7 \
  --deletion-protection

# Proxy for secure connection from GKE (no public IP needed)
# Run Cloud SQL Auth Proxy as a sidecar:
spec:
  containers:
  - name: cloudsql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.0
    args:
    - "--structured-logs"
    - "my-project:us-central1:prod-db"
    securityContext:
      runAsNonRoot: true
```

**Key concepts**:
- **HA**: Regional replication with automatic failover (60s typical)
- **Read replicas**: Offload read traffic
- **Point-in-time recovery**: Restore to any second within retention window
- **Private IP**: Access only within VPC (no public internet exposure)

---

## 3. Pub/Sub

Fully managed asynchronous messaging service.

```
Publisher ──► Topic ──► Subscription(s) ──► Subscriber(s)
                │
                ├──► sub-email-notifs    ──► email-service
                ├──► sub-audit-log      ──► audit-service
                └──► sub-analytics      ──► bigquery (push)

Features:
  - At-least-once delivery
  - Ordering (per key, within region)
  - Dead letter topics (failed messages)
  - Push (HTTP webhook) or Pull model
  - Replay / seek to timestamp
  - 7-day message retention max
```

```bash
# Create topic and subscription
gcloud pubsub topics create order-events
gcloud pubsub subscriptions create order-processor \
  --topic=order-events \
  --ack-deadline=60 \
  --message-retention-duration=7d \
  --dead-letter-topic=order-events-dlq \
  --max-delivery-attempts=5

# Publish message
gcloud pubsub topics publish order-events \
  --message='{"order_id":"123","status":"paid"}' \
  --attribute="type=order.paid,version=1"

# Pull messages
gcloud pubsub subscriptions pull order-processor --auto-ack --limit=10
```

---

## 4. Cloud Build (CI/CD alternative to Jenkins)

Serverless CI/CD. No infrastructure to manage.

```yaml
# cloudbuild.yaml
steps:
# Step 1: Run tests
- name: 'golang:1.21'
  entrypoint: 'go'
  args: ['test', './...', '-v']
  env:
  - 'CGO_ENABLED=0'

# Step 2: Build Docker image
- name: 'gcr.io/cloud-builders/docker'
  args:
  - 'build'
  - '-t'
  - 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA'
  - '-t'
  - 'gcr.io/$PROJECT_ID/myapp:latest'
  - '.'

# Step 3: Push to Artifact Registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA']

# Step 4: Deploy to GKE
- name: 'gcr.io/cloud-builders/kubectl'
  args:
  - 'set'
  - 'image'
  - 'deployment/myapp'
  - 'app=gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA'
  env:
  - 'CLOUDSDK_COMPUTE_ZONE=us-central1'
  - 'CLOUDSDK_CONTAINER_CLUSTER=my-cluster'

images:
- 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA'
- 'gcr.io/$PROJECT_ID/myapp:latest'

options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
```

**Triggers**: GitHub push, PR, manual. Connects via Cloud Build GitHub App.

---

## 5. Artifact Registry

Successor to GCR. Stores Docker images, npm packages, Maven/Gradle, Python packages.

```bash
# Create repository
gcloud artifacts repositories create myapp-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production Docker images" \
  --immutable-tags              # prevent overwriting existing tags

# Auth
gcloud auth configure-docker us-central1-docker.pkg.dev

# Push
docker tag myapp:v1 us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1
docker push us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1

# Vulnerability scanning (automatic on push)
gcloud artifacts docker images list us-central1-docker.pkg.dev/my-project/myapp-repo/myapp
gcloud artifacts docker images describe us-central1-docker.pkg.dev/.../myapp:v1 --show-package-vulnerability
```

---

## 6. Secret Manager

Centralized secret storage with versioning, rotation, and audit logging.

```bash
# Create secret
echo -n "supersecret" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy=automatic

# Add new version
echo -n "newpassword" | gcloud secrets versions add db-password --data-file=-

# Access
gcloud secrets versions access latest --secret=db-password

# Grant access to a service account
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:myapp-sa@project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**In GKE** — use External Secrets Operator:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: db-password           # creates a K8s Secret
  data:
  - secretKey: password
    remoteRef:
      key: db-password          # GCP Secret Manager name
      version: latest
```

---

## 7. Cloud Monitoring & Logging

```bash
# Query logs with gcloud
gcloud logging read \
  'resource.type="k8s_container" AND resource.labels.cluster_name="my-cluster" AND severity>=ERROR' \
  --project=my-project \
  --limit=50 \
  --format=json

# Create log-based metric
gcloud logging metrics create error-count \
  --description="Count of error logs" \
  --log-filter='resource.type="k8s_container" AND severity=ERROR'

# Create alert on log-based metric
# (via Cloud Console or Terraform)
```

### Log Sinks (export logs)
```bash
# Export all K8s logs to GCS for long-term archival
gcloud logging sinks create k8s-logs-archive \
  storage.googleapis.com/my-logs-bucket \
  --log-filter='resource.type="k8s_container"'

# Export to BigQuery for analysis
gcloud logging sinks create k8s-logs-bq \
  bigquery.googleapis.com/projects/my-project/datasets/k8s_logs \
  --log-filter='resource.type="k8s_container"'
```

---

## 8. Cloud IAM

Identity and Access Management. Who can do what on which resource.

```
Principal:  User | Service Account | Group | Domain
Role:       Predefined | Custom | Basic
Resource:   Project | Folder | Organization | GCS Bucket | etc.

Hierarchy:
  Organization
  └─ Folders (by team/product)
     └─ Projects
        └─ Resources

Inheritance: permissions set at higher levels flow down
```

```bash
# Grant role
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:myapp@my-project.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Create custom role
gcloud iam roles create customDeployer \
  --project=my-project \
  --title="Custom Deployer" \
  --permissions="container.deployments.update,container.pods.list,container.pods.get"

# Audit who has what
gcloud projects get-iam-policy my-project --format=json | \
  jq '.bindings[] | select(.role | contains("Owner"))'
```

---

## 9. VPC & Networking

```bash
# Create custom VPC (avoid default VPC in production)
gcloud compute networks create prod-vpc --subnet-mode=custom

# Create subnets
gcloud compute networks subnets create prod-subnet \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.0.0.0/20 \
  --secondary-range pods=10.4.0.0/14,services=10.0.32.0/20  # for GKE

# Firewall rules
gcloud compute firewall-rules create allow-internal \
  --network=prod-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

# Cloud NAT (allow private nodes to reach internet)
gcloud compute routers create prod-router --network=prod-vpc --region=us-central1
gcloud compute routers nats create prod-nat \
  --router=prod-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges

# VPC Peering (connect two VPCs)
gcloud compute networks peerings create peer-prod-to-data \
  --network=prod-vpc \
  --peer-project=data-project \
  --peer-network=data-vpc
```

---

## 10. Cloud Armor (WAF/DDoS)

```bash
# Create security policy
gcloud compute security-policies create myapp-policy \
  --description="WAF policy for myapp"

# Block specific countries
gcloud compute security-policies rules create 1000 \
  --security-policy=myapp-policy \
  --expression="origin.region_code == 'RU' || origin.region_code == 'CN'" \
  --action=deny-403

# Rate limiting
gcloud compute security-policies rules create 2000 \
  --security-policy=myapp-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=1000 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600

# OWASP rules (pre-built)
gcloud compute security-policies rules create 3000 \
  --security-policy=myapp-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403

# Attach to Backend Service (LB)
gcloud compute backend-services update myapp-backend \
  --security-policy=myapp-policy \
  --global
```

---

## Interview Questions — GCP (6 YOE Level)

**Q: What is the difference between Cloud Run and GKE?**
> **Cloud Run**: Serverless containers. Scales to 0. Pay per request. No cluster management. Good for HTTP services with variable/bursty traffic. Cold start latency. **GKE**: Full K8s. More control. Better for complex microservices, stateful workloads, long-running jobs, custom networking. Persistent cost (always-on nodes). Use Cloud Run for simpler services, GKE for complex orchestration needs.

**Q: How does IAM work in GCP and what are best practices?**
> IAM follows least-privilege: grant minimum permissions needed. Use predefined roles over basic roles (Owner/Editor are too broad). Prefer group membership over individual user grants. Use service accounts for workloads, not user accounts. Enable Workload Identity for GKE (no JSON keys). Audit with Cloud Audit Logs. Use organizational policies for guardrails.

**Q: How do you handle Pub/Sub message ordering and at-least-once delivery?**
> Enable message ordering (with ordering key). Design consumers to be **idempotent** (processing same message twice is safe). Use deduplication ID or check if already processed before acting. For failed messages: use dead letter topics with max delivery attempts. Implement exponential backoff in consumer.

**Q: Explain how you would architect a high-availability system on GCP.**
> Multi-zone GKE cluster (regional). Cloud SQL with HA (REGIONAL availability type). GCS for stateless storage (built-in HA). HTTPS LB with health checks (auto-remove unhealthy backends). Cloud Armor for DDoS protection. Cloud CDN for static assets. Multi-region Cloud Spanner for global DB. Alert on error budgets. Runbooks and chaos engineering to validate resilience.

**Q: What is VPC Service Controls?**
> A security perimeter around GCP services that prevents data exfiltration. Even if an attacker gets valid credentials, they cannot access data outside the perimeter (e.g., cannot copy BigQuery data to an external project). Applied at the organization/folder level. Essential for compliance (HIPAA, PCI) environments.
