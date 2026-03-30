# Ansible - Core Concepts, Interview Q&A & Scenarios
> Senior DevOps/SRE Level | Configuration Management Expert

---

## CORE CONCEPTS

### What is Ansible?
Ansible is an agentless, open-source configuration management, application deployment, and task automation tool.

**Key features**:
- **Agentless**: Uses SSH (Linux) or WinRM (Windows). No agent to install.
- **Idempotent**: Running same playbook multiple times produces same result.
- **YAML-based**: Human-readable playbooks.
- **Push-based**: Controller pushes config to managed nodes.

### Architecture
```
Ansible Controller (your machine or CI server)
    ↓ SSH
Managed Nodes (servers, VMs, network devices)
```

### Key Components
| Component | Description |
|---|---|
| **Inventory** | List of managed hosts (static or dynamic) |
| **Playbook** | YAML file with automation tasks |
| **Play** | A set of tasks for a group of hosts |
| **Task** | Single unit of action (run a module) |
| **Module** | Ansible built-in function (apt, yum, copy, template, service...) |
| **Role** | Reusable, structured automation package |
| **Handler** | Task triggered only when notified (e.g., restart nginx after config change) |
| **Variable** | Dynamic values (vars, vars_files, group_vars, host_vars) |
| **Fact** | Auto-discovered host information (OS, IP, RAM, etc.) |
| **Vault** | Encrypted secrets storage |

---

## PLAYBOOK STRUCTURE

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true  # Run as sudo
  vars:
    app_port: 8080
    max_connections: 1000

  pre_tasks:
  - name: Update package cache
    apt:
      update_cache: yes
      cache_valid_time: 3600

  tasks:
  - name: Install Nginx
    apt:
      name: nginx
      state: present

  - name: Deploy Nginx config
    template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: '0644'
      validate: '/usr/sbin/nginx -t -c %s'
    notify: Restart Nginx

  - name: Ensure Nginx is running
    service:
      name: nginx
      state: started
      enabled: yes

  handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```

---

## INVENTORY

### Static Inventory
```ini
[webservers]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11

[dbservers]
db1 ansible_host=10.0.2.10 ansible_user=ubuntu

[production:children]
webservers
dbservers

[production:vars]
ansible_ssh_private_key_file=~/.ssh/production.pem
```

### Dynamic Inventory (AWS/GCP)
```bash
# AWS dynamic inventory
ansible-inventory -i aws_ec2.yaml --list

# aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions: [us-east-1]
filters:
  tag:Environment: production
keyed_groups:
- prefix: tag
  key: tags
```

---

## ROLES

### Role Directory Structure
```
roles/
└── nginx/
    ├── defaults/      # Default variables (lowest priority)
    │   └── main.yml
    ├── vars/          # Fixed variables (higher priority)
    │   └── main.yml
    ├── tasks/         # Main task list
    │   └── main.yml
    ├── handlers/      # Handlers
    │   └── main.yml
    ├── templates/     # Jinja2 templates
    │   └── nginx.conf.j2
    ├── files/         # Static files
    ├── meta/          # Role metadata, dependencies
    │   └── main.yml
    └── README.md
```

---

## ANSIBLE VAULT

```bash
# Encrypt a file
ansible-vault encrypt secrets.yml

# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Run playbook with vault
ansible-playbook deploy.yml --ask-vault-pass
# Or use password file:
ansible-playbook deploy.yml --vault-password-file ~/.vault_pass

# Encrypt single string (for inline use)
ansible-vault encrypt_string 'mypassword' --name 'db_password'
```

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: What makes Ansible different from Puppet/Chef/Salt?

| Aspect | Ansible | Puppet | Chef | Salt |
|---|---|---|---|---|
| Agent | Agentless (SSH) | Agent required | Agent (chef-client) | Agent (minion) or agentless |
| Language | YAML | Puppet DSL | Ruby | YAML/Python |
| Execution | Push-based | Pull-based | Pull-based | Push + Pull |
| Ease | Simplest | Complex | Complex | Medium |
| Best for | Config mgmt, app deploy | Large fleet | Complex cookbooks | High-scale |

### Q2: What is idempotency in Ansible?
Idempotency means running the same playbook multiple times gives the same result. No unintended side effects.

Most Ansible modules are idempotent by default:
- `apt: name=nginx state=present` — won't reinstall if already installed
- `copy: src=app.conf dest=/etc/app.conf` — only copies if file differs

**Shell/command modules are NOT idempotent** — avoid or use `creates`/`removes`:
```yaml
- name: Initialize database (only if not done)
  command: /app/init-db.sh
  args:
    creates: /app/.db-initialized  # Only runs if this file doesn't exist
```

### Q3: What is the difference between vars, defaults, and group_vars?

**Variable precedence** (lower number = lower priority):
1. role defaults (defaults/main.yml)
2. group_vars/all
3. group_vars/specific_group
4. host_vars/specific_host
5. play vars
6. task vars
7. extra vars (-e flag) ← **Highest priority**

**defaults/main.yml**: Lowest priority — override-able everywhere. Use for sensible defaults.
**vars/main.yml**: Higher priority. For values that should be consistent within role.
**group_vars**: Environment-specific values.

### Q4: When would you use Ansible vs Terraform?

| Aspect | Ansible | Terraform |
|---|---|---|
| **Provision infra** | Yes (EC2, VPC) but less ideal | **Best choice** |
| **Configure OS/apps** | **Best choice** | No |
| **State management** | No state file | State file tracks infra |
| **Idempotency** | Module-level | Resource-level |

**Common pattern**: Terraform provisions EC2, Ansible configures the OS/app on it.

### Q5: How do handlers work and when to use them?
Handlers run only when notified by a task, and only once at end of play (even if notified multiple times).

```yaml
tasks:
- name: Update app config
  template:
    src: app.conf.j2
    dest: /etc/app/app.conf
  notify: Restart app  # Triggers handler if config changed

- name: Update app env
  copy:
    src: app.env
    dest: /etc/app/env
  notify: Restart app  # Also notifies — but handler runs ONCE

handlers:
- name: Restart app
  service:
    name: myapp
    state: restarted
```

### Q6: How do you handle rolling updates with Ansible?
```yaml
- hosts: webservers
  serial: 2  # Run on 2 hosts at a time (rolling)
  # serial: 25%  # Or 25% at a time
  max_fail_percentage: 20  # Stop if >20% fail

  tasks:
  - name: Remove from LB
    delegate_to: loadbalancer
    command: /usr/bin/remove-from-lb.sh {{ inventory_hostname }}

  - name: Deploy app
    # ... deploy tasks ...

  - name: Add back to LB
    delegate_to: loadbalancer
    command: /usr/bin/add-to-lb.sh {{ inventory_hostname }}
```

### Q7: How did you use Ansible at Altruist Technologies?
Real-world experience to talk about:

- **Server provisioning**: Automated setup of new EC2 instances — installing packages, configuring services, setting up users
- **Configuration management**: Managing Nginx configs, application configs across environments
- **Reduced manual setup by 60%**: Automated what used to be 2-hour manual setup into 10-minute playbook run
- **Idempotent deployments**: Could re-run safely on existing servers without issues
- **Ansible Vault**: Managed secrets (DB passwords, API keys) securely
- **Dynamic inventory**: Used AWS dynamic inventory to target EC2 by tags

---

## REAL-WORLD: EC2 Setup Playbook

```yaml
---
- name: Configure application server
  hosts: app_servers
  become: true

  roles:
  - common          # OS hardening, users, SSH config
  - docker          # Install and configure Docker
  - nginx           # Install and configure Nginx
  - app             # Deploy application
  - monitoring      # Install Node Exporter

# Run it:
# ansible-playbook -i inventory/production site.yml --tags app
```
