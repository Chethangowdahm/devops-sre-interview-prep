# Terraform - Core Concepts & Interview Q&A
> Senior DevOps/SRE Level | 6+ Years Experience

---

## CORE CONCEPTS

### What is Terraform?
Terraform is an open-source Infrastructure as Code (IaC) tool by HashiCorp. Uses HCL (HashiCorp Configuration Language) to define infrastructure in a declarative way.

**Key principles**: Declarative, idempotent, version-controlled infrastructure.

### Terraform Workflow
```
Write HCL → terraform init → terraform plan → terraform apply → terraform destroy
```

### Core Commands
```bash
terraform init          # Download providers, initialize backend
terraform validate      # Validate syntax
terraform fmt           # Format code
terraform plan          # Preview changes (dry run)
terraform apply         # Apply changes
terraform apply -auto-approve  # Skip confirmation
terraform destroy       # Destroy all resources
terraform show          # Show current state
terraform state list    # List resources in state
terraform state show aws_instance.web  # Show specific resource
terraform output        # Show output values
terraform import        # Import existing resource into state
terraform taint         # Mark resource for recreation
terraform refresh       # Sync state with real infrastructure
terraform workspace list/new/select  # Manage workspaces
```

---

## KEY CONCEPTS

### State
Terraform maintains a state file (terraform.tfstate) that maps your configuration to real infrastructure.
- **Local state**: Default, stored locally. NOT suitable for teams.
- **Remote state**: S3 + DynamoDB (AWS), GCS (GCP), Terraform Cloud. Required for teams.

```hcl
# Remote state backend (AWS S3)
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/eks/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # State locking
  }
}
```

### State Locking
Prevents concurrent state modifications. AWS uses DynamoDB; GCS has native locking.

### Providers
Plugins that interact with cloud APIs (AWS, GCP, Azure, Kubernetes, Datadog...).

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Compatible with 5.x
    }
    google = {
      source  = "hashicorp/google"
      version = ">= 4.0, < 6.0"
    }
  }
  required_version = ">= 1.5.0"
}
```

### Resources
Infrastructure components you want to create/manage.

### Data Sources
Read-only reference to existing infrastructure (not managed by Terraform).

```hcl
# Data source: fetch existing VPC
data "aws_vpc" "existing" {
  tags = {
    Name = "production-vpc"
  }
}

resource "aws_subnet" "new" {
  vpc_id = data.aws_vpc.existing.id  # Reference data source
  cidr_block = "10.0.100.0/24"
}
```

### Variables
```hcl
variable "instance_type" {
  type        = string
  default     = "t3.medium"
  description = "EC2 instance type"
  validation {
    condition     = contains(["t3.medium", "t3.large", "t3.xlarge"], var.instance_type)
    error_message = "Must be t3.medium, t3.large, or t3.xlarge"
  }
}

# Usage
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

### Outputs
```hcl
output "cluster_endpoint" {
  value     = module.eks.cluster_endpoint
  sensitive = false
}
```

### Modules
Reusable, encapsulated infrastructure components.
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  name    = "production-vpc"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = false  # HA: one per AZ
}
```

### Workspaces
Separate state files for different environments within same config.

```bash
terraform workspace new staging
terraform workspace select production
terraform workspace list
```

**When to use**: Simple env separation. For complex setups, separate directories/repos are often better.

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: terraform plan vs terraform apply?
**plan**: Dry run. Shows what WILL be created/modified/destroyed. No actual changes. Always review before apply.
**apply**: Executes the plan. Makes actual API calls to create/modify/destroy infrastructure.

### Q2: What is Terraform state and why is it important?
State tracks the mapping between Terraform config and real-world resources.
- Without state: Terraform can't know what already exists
- With state: Terraform can calculate the diff (plan) and manage drift
- Remote state is mandatory for teams to prevent conflicts

### Q3: How do you handle Terraform state in a team environment?
1. **Remote backend** (S3 + DynamoDB for AWS)
2. **State locking** (DynamoDB prevents concurrent runs)
3. **Encrypted state** (contains sensitive data)
4. **Access controls** (IAM policies for state bucket)
5. **State versioning** (S3 versioning for recovery)
6. **CI/CD only** applies: Never run terraform locally in production

### Q4: What happens if you manually change a resource that Terraform manages?
This is called **configuration drift**. On next `terraform plan`, Terraform detects the diff and will show it wants to revert to desired state.
- `terraform plan`: Shows drift
- `terraform apply`: Fixes drift by reverting to IaC state
- `terraform refresh`: Updates state file to match reality (without making changes)

**Prevention**: Use Terraform Sentinel (Terraform Cloud) or CI checks to enforce IaC-only changes. Use AWS Config/GCP Asset Inventory for drift detection alerts.

### Q5: What is terraform import?
Imports existing infrastructure into Terraform state WITHOUT recreating it.

```bash
# 1. Write the resource config in HCL first
# 2. Import
terraform import aws_s3_bucket.existing my-existing-bucket
terraform import aws_instance.web i-1234567890abcdef0
```

**Use case**: When you have pre-existing infrastructure not managed by Terraform.

### Q6: count vs for_each?
```hcl
# count: Creates N identical resources (index-based)
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.medium"
  # Reference: aws_instance.web[0], aws_instance.web[1]
}

# for_each: Creates resources from map/set (key-based, more stable)
resource "aws_s3_bucket" "logs" {
  for_each = toset(["app-logs", "access-logs", "audit-logs"])
  bucket   = each.key
  # Reference: aws_s3_bucket.logs["app-logs"]
}
```

**Prefer for_each**: Removing middle item with count shifts indices, causing recreation. for_each uses stable keys.

### Q7: How do you structure Terraform for multiple environments?

**Option A: Directory structure (recommended)**:
```
terraform/
├── modules/
│   ├── eks/
│   ├── vpc/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
```

**Option B: Workspaces** (simpler but limited):
```bash
terraform workspace select production
terraform apply -var-file=production.tfvars
```

**Option C: Terragrunt** (DRY wrapper, manages remote state per env automatically).

### Q8: What are Terraform provisioners? Should you use them?
Provisioners execute scripts on resources after creation (local-exec, remote-exec, file).

**Avoid if possible**: Break idempotency, hard to test, fail silently.
**Better alternatives**: User data scripts (EC2), Ansible, cloud-init, baked AMIs.

### Q9: How do you prevent accidental destruction of critical resources?
```hcl
resource "aws_rds_instance" "production" {
  lifecycle {
    prevent_destroy = true  # Terraform will error if you try to destroy
    create_before_destroy = true  # For zero-downtime replacements
    ignore_changes = [
      tags,  # Don't revert tag changes made outside Terraform
      instance_class  # Allow manual scaling without Terraform reverting
    ]
  }
}
```

### Q10: What is the difference between terraform.tfvars and variables.tf?
- **variables.tf**: Declares variables (name, type, description, validation, default)
- **terraform.tfvars**: Assigns values to variables for a specific environment
- **Auto-loaded**: terraform.tfvars, *.auto.tfvars loaded automatically
- **Explicit**: -var-file=custom.tfvars or -var key=value

### Q11: Explain Terraform's dependency graph
Terraform builds a DAG (Directed Acyclic Graph) of all resources and their dependencies.
- Explicit dependency: Using resource outputs as inputs
- Implicit dependency: depends_on argument
- Parallel creation: Independent resources created in parallel

```hcl
# Explicit dependency (preferred)
resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id  # depends on aws_subnet.private
  route_table_id = aws_route_table.private.id
}

# Implicit dependency
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy_attachment.web_policy]
}
```

### Q12: How do you handle Terraform in CI/CD?
```yaml
# GitHub Actions Terraform Pipeline
- name: Terraform Plan
  run: |
    terraform init -backend-config=backend-production.hcl
    terraform validate
    terraform plan -out=tfplan -var-file=production.tfvars

- name: Post Plan to PR
  uses: actions/github-script@v6
  # Post terraform plan output as PR comment

- name: Terraform Apply (main branch only)
  if: github.ref == 'refs/heads/main'
  run: terraform apply tfplan
```

**Best practices**:
- Plan on PR, apply on merge to main
- Use OIDC/Workload Identity (no static credentials)
- Lock provider versions
- Run terraform fmt and validate in CI
