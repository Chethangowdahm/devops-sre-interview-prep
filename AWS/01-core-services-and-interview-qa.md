# AWS - Core Services & Interview Q&A
> Senior DevOps/SRE Level | Cloud Infrastructure Expert

---

## CORE AWS SERVICES

### Compute
| Service | Use Case |
|---|---|
| **EC2** | Virtual machines. Full control over OS. |
| **EKS** | Managed Kubernetes. Runs control plane, you manage workers. |
| **ECS** | AWS-native container orchestration (simpler than EKS). |
| **Lambda** | Serverless functions. Pay per invocation. |
| **Auto Scaling Group** | Auto-scale EC2 instances based on demand. |

### Storage
| Service | Use Case |
|---|---|
| **S3** | Object storage. Static websites, backups, data lake. |
| **EBS** | Block storage for EC2. SSD/HDD. AZ-specific. |
| **EFS** | Shared NFS filesystem across AZs. |
| **Glacier** | Archive storage. Very cheap, slow retrieval. |

### Database
| Service | Use Case |
|---|---|
| **RDS** | Managed relational DB (MySQL, PostgreSQL, etc.) |
| **Aurora** | AWS-native MySQL/PostgreSQL, 5x faster, serverless option |
| **DynamoDB** | Serverless NoSQL. Single-digit ms latency at any scale. |
| **ElastiCache** | Managed Redis/Memcached. Caching layer. |

### Networking
| Service | Use Case |
|---|---|
| **VPC** | Virtual private network. Isolated AWS environment. |
| **Route 53** | DNS service. Also health checks, routing policies. |
| **CloudFront** | CDN. Edge caching for static/dynamic content. |
| **ELB** | Load balancers (ALB=HTTP, NLB=TCP, CLB=legacy) |
| **Transit Gateway** | Connect multiple VPCs and on-premises. |
| **VPC Peering** | Connect two VPCs privately. |

### Security & Identity
| Service | Use Case |
|---|---|
| **IAM** | Users, roles, policies. Authentication + authorization. |
| **Secrets Manager** | Store/rotate secrets. API keys, passwords. |
| **KMS** | Encryption key management. |
| **Security Groups** | Stateful firewall at instance level. |
| **NACLs** | Stateless firewall at subnet level. |
| **WAF** | Web Application Firewall. SQL injection, XSS protection. |

### Monitoring
| Service | Use Case |
|---|---|
| **CloudWatch** | Metrics, logs, alarms, dashboards. |
| **CloudTrail** | API audit log. Who did what, when. |
| **AWS Config** | Resource configuration history and compliance. |
| **X-Ray** | Distributed tracing for applications. |

---

## VPC DEEP DIVE

### VPC Architecture
```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24) [AZ-1a] ← Internet accessible
│   ├── NAT Gateway
│   ├── Bastion Host (or SSM)
│   └── Load Balancer
├── Private Subnet (10.0.2.0/24) [AZ-1a] ← Internal only
│   ├── EC2 / EKS nodes
│   └── Connects to internet via NAT Gateway
└── Database Subnet (10.0.3.0/24) [AZ-1a] ← DB only
    └── RDS instances

Internet Gateway → Public Subnet
Private Subnet → NAT Gateway → Internet Gateway
```

### Security Groups vs NACLs
| Aspect | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must explicitly allow return) |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |

---

## IAM DEEP DIVE

### Key Concepts
- **User**: Human identity with long-term credentials
- **Role**: Assumed identity, short-term credentials (preferred)
- **Group**: Collection of users
- **Policy**: JSON document defining permissions

### IAM Policy Structure
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### IRSA (IAM Roles for Service Accounts)
Allows EKS pods to assume IAM roles. No static credentials needed.

```yaml
# ServiceAccount with IAM role annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
```

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: What is the difference between ALB and NLB?

| Aspect | ALB (Application LB) | NLB (Network LB) |
|---|---|---|
| Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP) |
| Routing | Path, host, header-based | IP/port-based |
| Protocols | HTTP, HTTPS, WebSocket | TCP, UDP, TLS |
| Performance | ~100ms latency | Sub-millisecond |
| Use case | Microservices, HTTP apps | Game servers, IoT, ultra-low latency |
| SSL Termination | Yes | Yes |
| Static IP | No (uses DNS) | Yes (Elastic IP) |

### Q2: Explain S3 storage classes and cost optimization

| Class | Use Case | Retrieval |
|---|---|---|
| S3 Standard | Frequently accessed | Immediate |
| S3 Intelligent-Tiering | Unknown/changing access | Immediate |
| S3 Standard-IA | Infrequent access | Immediate |
| S3 Glacier Instant | Archives, immediate retrieval | Milliseconds |
| S3 Glacier Flexible | Archives | 1-12 hours |
| S3 Glacier Deep Archive | Long-term archive | 12-48 hours |

**Lifecycle policy example**:
```json
{
  "Rules": [{
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"}
    ],
    "Expiration": {"Days": 365}
  }]
}
```

### Q3: How do you set up high availability in AWS?
1. **Multi-AZ**: Spread across 3 AZs minimum
2. **Load Balancer**: ALB/NLB in front of EC2/EKS
3. **Auto Scaling**: Min 2 instances per AZ
4. **RDS Multi-AZ**: Synchronous replication to standby
5. **Read Replicas**: Scale reads horizontally
6. **Route 53 health checks**: Failover DNS routing
7. **CloudFront**: Edge caching reduces origin load

### Q4: What is a NAT Gateway and when do you need it?
NAT Gateway allows instances in private subnets to initiate outbound connections to the internet while preventing inbound connections.

**Use case**: Private EC2/EKS nodes need to pull Docker images, access S3, download packages.

**HA setup**: One NAT Gateway per AZ. Route table for each AZ's private subnet points to its NAT Gateway.

**Cost tip**: NAT Gateway charges per GB. Use VPC endpoints (PrivateLink) for S3/DynamoDB to avoid NAT costs.

### Q5: How does EKS differ from self-managed Kubernetes?

| Aspect | EKS | Self-managed |
|---|---|---|
| Control plane | AWS manages | You manage |
| Upgrades | Managed | Manual |
| HA | Built-in | Must configure |
| Integration | Native AWS IAM, LB, EBS | Manual integration |
| Cost | $0.10/hr per cluster + nodes | EC2 costs only |

**EKS node types**:
- **Managed Node Groups**: EC2 instances, AWS manages patching/updates
- **Fargate**: Serverless pods, no node management
- **Self-managed nodes**: Full control, more maintenance

### Q6: What is CloudWatch and how do you use it effectively?
CloudWatch provides: Metrics, Logs, Alarms, Dashboards, Events, Insights.

```bash
# Create alarm for high CPU
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123:my-topic
```

**CloudWatch Insights for log analysis**:
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) by bin(5m)
| sort @timestamp desc
| limit 100
```

### Q7: Explain AWS IAM best practices
1. **Root account**: Enable MFA, never use for daily work, no access keys
2. **Least privilege**: Grant minimum required permissions
3. **Use roles, not users**: For services/applications
4. **IRSA for EKS**: Pod-level IAM permissions without sharing credentials
5. **SCPs (Service Control Policies)**: Org-level guardrails in AWS Organizations
6. **Access Analyzer**: Detect overly permissive policies
7. **Rotate credentials**: Automate key rotation with Secrets Manager

### Q8: How do you reduce AWS costs?
1. **Right-sizing**: Use CloudWatch + Cost Explorer to find overprovisioned resources
2. **Reserved Instances / Savings Plans**: 40-70% discount for committed usage
3. **Spot Instances**: 70-90% discount for fault-tolerant workloads (K8s batch jobs)
4. **S3 Lifecycle policies**: Move to cheaper tiers
5. **NAT Gateway**: Use VPC endpoints for S3/DynamoDB
6. **Delete unused resources**: Unattached EBS, old snapshots, idle LBs
7. **Auto Scaling**: Scale down during off-hours
8. **Data transfer optimization**: Keep traffic within same region/AZ

---

## SCENARIO: EKS Pod Can't Access S3
```bash
# Symptoms: aws s3 ls fails from pod
# Diagnosis:

# 1. Check pod's service account
kubectl describe pod <pod> | grep ServiceAccount

# 2. Check service account annotation
kubectl describe sa <sa-name> -n <ns>
# Should have: eks.amazonaws.com/role-arn annotation

# 3. Check IAM role trust policy
aws iam get-role --role-name my-app-role
# Trust policy must include EKS OIDC provider + service account

# 4. Check IAM role permissions
aws iam list-attached-role-policies --role-name my-app-role

# 5. Check IRSA is working
kubectl exec -it <pod> -- aws sts get-caller-identity
# Should show role ARN, not EC2 instance profile
```
