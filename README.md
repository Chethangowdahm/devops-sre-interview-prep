# SRE & DevOps Interview Prep — 6 Years Experience
> Complete guide covering Docker, Kubernetes, GKE, Jenkins, Git/GitHub, Prometheus, Grafana, Datadog, and GCP services.

---

## Folder Structure

```
interview-prep/
├── README.md                  ← You are here (overview + master workflow)
├── 01-docker.md               ← Docker deep dive
├── 02-kubernetes.md           ← Kubernetes deep dive
├── 03-gke.md                  ← Google Kubernetes Engine
├── 04-jenkins.md              ← Jenkins CI/CD
├── 05-git-github.md           ← Git & GitHub workflows
├── 06-prometheus-grafana.md   ← Prometheus + Grafana monitoring
├── 07-datadog.md              ← Datadog observability
├── 08-gcp-services.md         ← Key GCP services for SRE/DevOps
├── 09-end-to-end-workflow.md  ← Full E2E workflow walkthrough
└── 10-sre-concepts.md         ← SRE principles, SLOs, SLAs, incident mgmt
```

---

## Master Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEVELOPER WORKSTATION                               │
│   Code Editor  →  git commit  →  git push  →  GitHub Pull Request          │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │  Webhook trigger
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CI/CD LAYER (Jenkins)                              │
│                                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │  Checkout│→  │  Build   │→  │   Test   │→  │  Docker  │→  │  Push   │ │
│  │   Code   │   │(compile) │   │(unit/int)│   │  Build   │   │ to GCR  │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └─────────┘ │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │  Deploy trigger
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CONTAINER REGISTRY (GCR / Artifact Registry)            │
│              docker images tagged with git SHA / version                    │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │  kubectl apply / helm upgrade
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER (GKE)                                 │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CONTROL PLANE (Master)                                             │   │
│  │  API Server │ etcd │ Scheduler │ Controller Manager                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │                     │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │                     │
│  │  │Pod(app)│  │  │  │Pod(app)│  │  │  │Pod(app)│  │                     │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │                     │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │                     │
│  │  │kubelet │  │  │  │kubelet │  │  │  │kubelet │  │                     │
│  │  │kube-px │  │  │  │kube-px │  │  │  │kube-px │  │                     │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│                                                                             │
│  Services: LoadBalancer │ Ingress (nginx/GCE) │ HPA │ PDB                  │
│  Storage:  PersistentVolumes │ ConfigMaps │ Secrets                         │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │  Metrics & Logs
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     OBSERVABILITY LAYER                                     │
│                                                                             │
│  ┌──────────────────────┐     ┌──────────────────────┐                     │
│  │   PROMETHEUS STACK   │     │      DATADOG           │                   │
│  │  ┌────────────────┐  │     │  ┌──────────────────┐  │                   │
│  │  │  Prometheus    │  │     │  │  Datadog Agent   │  │                   │
│  │  │  (scrapes /    │  │     │  │  (DaemonSet)     │  │                   │
│  │  │   metrics)     │  │     │  └────────┬─────────┘  │                   │
│  │  └───────┬────────┘  │     │           │ sends to    │                   │
│  │          │            │     │  ┌────────▼─────────┐  │                   │
│  │  ┌───────▼────────┐  │     │  │  Datadog Cloud   │  │                   │
│  │  │  Alertmanager  │  │     │  │  (APM, Logs,     │  │                   │
│  │  └───────┬────────┘  │     │  │   Dashboards,    │  │                   │
│  │          │            │     │  │   Alerts)        │  │                   │
│  │  ┌───────▼────────┐  │     │  └──────────────────┘  │                   │
│  │  │    Grafana     │  │     └──────────────────────────┘                 │
│  │  │  (Dashboards)  │  │                                                   │
│  │  └────────────────┘  │                                                   │
│  └──────────────────────┘                                                   │
│                                                                             │
│  Cloud Logging (GCP) │ Cloud Monitoring │ Error Reporting                   │
└─────────────────────────────────────────────────────────────────────────────┘
                               │  Alerts
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     INCIDENT MANAGEMENT                                     │
│   PagerDuty / OpsGenie → On-call Engineer → Runbook → Postmortem           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference — Tool Purpose

| Tool             | Layer         | Primary Purpose                                      |
|-----------------|---------------|------------------------------------------------------|
| Git             | SCM           | Version control, branching, history                  |
| GitHub          | SCM           | Code hosting, PRs, code review, Actions              |
| Docker          | Container     | Package app + deps into portable image               |
| Kubernetes      | Orchestration | Run, scale, self-heal containers at scale            |
| GKE             | Managed K8s   | Google-managed K8s control plane on GCP              |
| Jenkins         | CI/CD         | Automate build, test, deploy pipelines               |
| Prometheus      | Monitoring    | Pull-based metrics collection and alerting           |
| Grafana         | Visualization | Dashboards on top of Prometheus/other data sources   |
| Datadog         | Observability | APM, logs, metrics, tracing in SaaS platform         |
| GCP             | Cloud         | Infrastructure: GKE, GCS, CloudSQL, PubSub, etc.     |

---

## Study Order

1. `05-git-github.md` → Foundation for everything
2. `01-docker.md` → Build artifacts
3. `04-jenkins.md` → Automate the pipeline
4. `02-kubernetes.md` → Run in production
5. `03-gke.md` → GCP-specific K8s
6. `06-prometheus-grafana.md` → Monitoring
7. `07-datadog.md` → Observability platform
8. `08-gcp-services.md` → Cloud ecosystem
9. `09-end-to-end-workflow.md` → Tie it all together
10. `10-sre-concepts.md` → SRE mindset & interviews
