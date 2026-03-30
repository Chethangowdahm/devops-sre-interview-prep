# DevOps & SRE Interview Preparation Guide
> Chethan H M | Senior DevOps/SRE Engineer | 6.2 Years Experience
> Bangalore, India | chethangowdahm318@gmail.com

---

## About This Repository

Complete interview preparation material for Senior DevOps and SRE roles.
Each tool has its own folder with: core concepts, interview Q&A, scenario-based questions, and real-world examples.

**Target**: Senior DevOps/SRE Engineer roles (6+ years experience)
**Coverage**: All tools and technologies from my resume

---

## Repository Structure

### Tool-Specific Folders

| Folder | Contents |
|---|---|
| [Kubernetes/](./Kubernetes/) | Core concepts, 16+ interview Q&A, 8 production scenarios, real-world examples |
| [Docker/](./Docker/) | Dockerfile best practices, networking, security, 12 interview Q&A |
| [Terraform/](./Terraform/) | IaC concepts, state management, 12 interview Q&A, CI/CD integration |
| [ArgoCD/](./ArgoCD/) | GitOps principles, sync strategies, RBAC, real-world scenarios |
| [Jenkins/](./Jenkins/) | Declarative pipelines, K8s agents, shared libraries, interview Q&A |
| [GitHub-Actions/](./GitHub-Actions/) | Workflows, OIDC, caching, reusable workflows, interview Q&A |
| [AWS/](./AWS/) | Core services, VPC, IAM, EKS, cost optimization, interview Q&A |
| [GCP/](./GCP/) | GKE, Workload Identity, Cloud Run, production Terraform setup |
| [Ansible/](./Ansible/) | Playbooks, roles, vault, idempotency, interview Q&A |
| [Monitoring-Observability/](./Monitoring-Observability/) | Prometheus, Grafana, Datadog, SLOs, PromQL, alerting |
| [Nginx-Linux-Bash/](./Nginx-Linux-Bash/) | Nginx config, Linux troubleshooting, Bash scripting |

### Interview Preparation Folder

| File | Contents |
|---|---|
| [01-HR-and-behavioral-questions.md](./Interview-Preparation/01-HR-and-behavioral-questions.md) | Tell me about yourself, STAR stories, salary negotiation, pre-interview checklist |
| [02-mock-technical-interview-modern-questions.md](./Interview-Preparation/02-mock-technical-interview-modern-questions.md) | Full mock interview with model answers, system design, GitOps, Platform Engineering |
| [03-SRE-principles-and-system-design.md](./Interview-Preparation/03-SRE-principles-and-system-design.md) | SRE principles, incident management, system design examples, quick-reference numbers |

---

## Key Topics Covered

### Kubernetes
- Architecture (control plane, worker nodes, etcd)
- All core objects (Pod, Deployment, StatefulSet, DaemonSet, Service, Ingress)
- Networking (CNI, NetworkPolicy, DNS)
- Scheduling (taints, affinity, resource management)
- Autoscaling (HPA, VPA, KEDA, Cluster Autoscaler)
- RBAC, Secrets management, Health probes
- Production scenarios with full solutions

### GitOps & CI/CD
- ArgoCD: sync strategies, ApplicationSets, RBAC, hooks
- Jenkins: declarative pipelines, K8s agents, shared libraries
- GitHub Actions: OIDC auth, caching, matrix builds, environment protection
- GitOps vs traditional CI/CD tradeoffs

### Cloud (AWS + GCP)
- AWS: VPC, EKS, IAM/IRSA, S3, RDS, monitoring
- GCP: GKE Autopilot, Workload Identity, Cloud Run, Terraform setup
- Cost optimization strategies
- Security best practices

### Monitoring & Observability
- Prometheus: PromQL, alerting rules, ServiceMonitor
- Grafana: dashboards, RED/USE methods
- Datadog: APM, SLO tracking, monitors
- SLI/SLO/SLA/Error budgets
- Distributed tracing

---

## Study Plan

**Week 1**: Core tools (Kubernetes, Docker, Terraform)
**Week 2**: Cloud platforms (AWS, GCP) + ArgoCD + GitOps
**Week 3**: CI/CD (Jenkins, GitHub Actions) + Monitoring
**Week 4**: Interview preparation + mock interviews + behavioral

**Daily practice**: Pick one scenario from any folder and explain it aloud as you would in an interview.

---

## Quick Tips for Interview Success

1. **Be specific**: Use numbers and real examples from your experience at Vendasta/Altruist
2. **Show depth**: Mention alternatives and tradeoffs — this demonstrates senior thinking
3. **Think aloud**: For system design, narrate your thinking process
4. **Admit gaps gracefully**: 'I haven't used X directly, but I'd approach it like Y'
5. **Use STAR method**: Situation, Task, Action, Result for behavioral questions
6. **Ask smart questions**: Prepare 3-4 questions that show strategic thinking

---

**Good luck, Chethan! You have 6.2 years of real production experience — trust it.**
