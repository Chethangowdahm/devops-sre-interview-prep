# Complete Interview Session — File 6: IaC — Terraform & Ansible Round

> **Round Type:** Infrastructure as Code deep dive
> **Your Edge:** Terraform daily at Vendasta for GCP. Ansible at Altruist for config management.

---

## SECTION 1: TERRAFORM

---

### Q1. Explain Terraform core workflow.

**[YOUR ANSWER]:**

Terraform workflow: Write → Init → Plan → Apply → Destroy

**terraform init:** Downloads provider plugins, initializes backend (GCS/S3), downloads modules.

**terraform plan:** Reads state, calls provider APIs for current state, computes diff — shows without changing.

**terraform apply:** Executes the plan, makes actual API calls, updates state file.

**State file:** JSON recording current state of all managed resources. Without it, Terraform doesn't know what it manages.

**At Vendasta:** State stored in GCS with versioning enabled. State locking prevents concurrent applies.

---

### Q2. What is Terraform state and why is remote state important?

**[YOUR ANSWER]:**

Terraform state maps configuration to real-world resources — tracking resource IDs, attributes, dependencies.

**Remote state benefits:**
- Team collaboration: Multiple engineers apply safely
- State locking: Prevents concurrent applies (race conditions)
- Versioning: Recover from accidental state corruption
- Security: Local state files contain sensitive data

```hcl
terraform {
  backend "gcs" {
    bucket  = "vendasta-terraform-state"
    prefix  = "production/gke-cluster"
  }
}
```

**Danger zone:** `terraform state rm` removes a resource from state without destroying it — used when migrating ownership between workspaces.

---

### Q3. Explain Terraform modules and design principles.

**[YOUR ANSWER]:**

Modules are reusable, parameterized Terraform configurations — equivalent of functions in programming.

```
modules/
  gke-cluster/
    main.tf       # Resource definitions
    variables.tf  # Input variables
    outputs.tf    # Output values
    versions.tf   # Provider versions
```

```hcl
module "gke_production" {
  source = "./modules/gke-cluster"
  project_id   = var.project_id
  region       = "us-central1"
  cluster_name = "prod-cluster"
  node_count   = 5
  machine_type = "n2-standard-4"
}
```

**Design principles:** Single responsibility, sensible defaults, clear inputs/outputs, version pinning.

**At Vendasta:** Module library covers GKE, Cloud SQL, GCS, VPC. New teams onboard infrastructure in under 30 minutes.

---

### Q4. terraform taint vs terraform import?

**[YOUR ANSWER]:**

**terraform replace (taint replacement):** Marks resource for destroy + re-creation.
```bash
terraform apply -replace='module.gke_production.google_container_node_pool.main'
```

**terraform import:** Brings existing real-world resources under Terraform management.
```bash
terraform import google_container_cluster.main projects/vendasta-prod/locations/us-central1/clusters/prod-cluster
```

**Real scenario:** A GKE cluster was created manually via console before adopting Terraform. I imported it, wrote matching HCL, and from that point it was fully Terraform-managed.

---

### Q5. Scenario: terraform apply failed halfway. How do you handle it?

**[YOUR ANSWER]:**

A partial apply leaves state partially applied. Terraform records what succeeded.

**Steps:**
1. Don't panic — Terraform recorded what succeeded
2. Run `terraform plan` again — shows only what remains
3. Fix the root cause error
4. Run `terraform apply` again — idempotent, already-created resources won't re-create

**If state corrupted:**
```bash
terraform state list                  # List all resources
terraform state show <resource>       # Show details
terraform state rm <resource>         # Remove from state
terraform import <resource> <id>      # Re-import
```

**Prevention:** Always `terraform plan` before apply. Use CI pipeline — never manual applies in production.

---

## SECTION 2: ANSIBLE

---

### Q6. Ansible vs Terraform — what's the difference?

**[YOUR ANSWER]:**

**Terraform:** Declarative, focused on PROVISIONING infrastructure — creates cloud resources, manages their lifecycle.

**Ansible:** Procedural, focused on CONFIGURATION MANAGEMENT — installs software, configures files, manages services on existing servers. Agentless (SSH).

**They complement each other:**
- Terraform creates the EC2 instance
- Ansible configures it — installs Nginx, deploys app, manages services

**At Altruist:** Terraform provisioned EC2 instances. Ansible configured them — Docker, Nginx, systemd services, SSH keys. Zero manual SSH needed.

---

### Q7. Show me a real Ansible playbook.

**[YOUR ANSWER]:**

```yaml
---
- name: Configure web application servers
  hosts: webservers
  become: yes
  vars:
    app_version: "{{ lookup('env', 'APP_VERSION') }}"
    app_port: 8080
  tasks:
    - name: Install packages
      apt:
        name: [docker.io, nginx, python3-pip]
        state: present
        update_cache: yes

    - name: Start Docker
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Deploy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Reload Nginx

    - name: Start application container
      docker_container:
        name: myapp
        image: gcr.io/myproject/myapp:{{ app_version }}
        state: started
        restart_policy: always
        ports: ["{{ app_port }}:8080"]

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

**Key concepts:** become for sudo, template for Jinja2 config files, notify/handlers for triggered actions, idempotency — running twice is safe.

**At Altruist:** Used this pattern to deploy across 10 EC2 instances in parallel. Deployment time dropped from 40 min (manual SSH) to 4 minutes.

---

### Q8. What are Ansible roles?

**[YOUR ANSWER]:**

Roles organize playbooks into reusable components with standard directory structure:

```
roles/
  nginx/
    tasks/main.yml       # Main task list
    handlers/main.yml    # Handlers
    templates/           # Jinja2 templates
    files/               # Static files
    vars/main.yml        # Variables
    defaults/main.yml    # Default variables
    meta/main.yml        # Dependencies
```

**Benefits:** Reusability, separation of concerns, independent testing with Molecule, sharing via Ansible Galaxy.

**At Altruist:** Built roles for nginx, docker, monitoring-agent, base-security-hardening. Every server gets base-security-hardening. New services get relevant roles applied.

---

### Q9. How do you manage secrets in Ansible?

**[YOUR ANSWER]:**

Using Ansible Vault — encrypts sensitive variables and files at rest.

```bash
# Create encrypted vars file
ansible-vault create group_vars/production/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/production/vault.yml

# Run playbook with vault
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
```

```yaml
# vault.yml content (encrypted at rest)
vault_db_password: 'SuperSecurePassword123'
vault_api_key: 'sk-xxxxxxxxxxx'
```

```yaml
# Reference in playbook
vars_files:
  - group_vars/production/vault.yml
tasks:
  - name: Configure database
    template:
      src: db.conf.j2  # Uses {{ vault_db_password }}
      dest: /etc/app/db.conf
```

**Best practice:** Store vault password in AWS Secrets Manager or HashiCorp Vault — retrieve in CI pipeline. Never in Git.

---

*Next: Move to File 07 — Monitoring & Observability Round*
