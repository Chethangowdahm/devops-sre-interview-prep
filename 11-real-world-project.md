# Real-World Project: E-Commerce Order Platform
## All Tools Working Together — End to End

> **Project**: "ShopFast" — a microservices-based e-commerce platform
> **Stack**: Go microservices + GKE + Jenkins + GitHub + Prometheus + Grafana + Datadog + GCP
> **Scale**: 50k daily active users, 500 orders/minute at peak
> **Team**: 8 engineers, 2 SREs

---

## 1. Application Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        SHOPFAST — SYSTEM ARCHITECTURE                            │
│                                                                                  │
│                              INTERNET                                            │
│                                  │                                               │
│                                  ▼                                               │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                    GCP CLOUD ARMOR (WAF + DDoS)                           │  │
│  │  Rules: block SQLi, XSS, rate-limit 1000 req/min per IP                  │  │
│  └───────────────────────────────────┬───────────────────────────────────────┘  │
│                                      │                                           │
│  ┌───────────────────────────────────▼───────────────────────────────────────┐  │
│  │           GCP HTTPS LOAD BALANCER (Global, Anycast IP)                    │  │
│  │  SSL termination │ Cloud CDN (static assets) │ Health checks              │  │
│  └──────┬──────────────────────┬────────────────────────┬─────────────────────┘  │
│         │                      │                        │                        │
│         ▼                      ▼                        ▼                        │
│  ┌─────────────┐  ┌────────────────────┐  ┌─────────────────────────┐          │
│  │  web-svc    │  │   api-gateway-svc   │  │   admin-svc             │          │
│  │ (React SPA) │  │   (Go, main entry)  │  │   (internal only)       │          │
│  │  /          │  │   /api/*            │  │   /admin/*              │          │
│  └─────────────┘  └────────┬───────────┘  └─────────────────────────┘          │
│                             │                                                    │
│         ┌───────────────────┼────────────────────────────────┐                  │
│         │                   │                                │                  │
│         ▼                   ▼                                ▼                  │
│  ┌─────────────┐   ┌──────────────────┐           ┌──────────────────┐         │
│  │  user-svc   │   │   order-svc      │           │  inventory-svc   │         │
│  │  (Go)       │   │   (Go) ← CORE    │           │  (Go)            │         │
│  │  port: 8081 │   │   port: 8082     │           │  port: 8083      │         │
│  │             │   │                  │           │                  │         │
│  │  ● Login    │   │  ● Create order  │           │  ● Stock check   │         │
│  │  ● Profile  │   │  ● Track status  │           │  ● Reserve item  │         │
│  │  ● Auth JWT │   │  ● Cancel        │           │  ● Release stock │         │
│  └──────┬──────┘   └────────┬─────────┘           └────────┬─────────┘         │
│         │                   │                               │                   │
│         │         ┌─────────┼───────────────────┐           │                   │
│         │         │         │                   │           │                   │
│         │         ▼         ▼                   ▼           │                   │
│         │  ┌───────────┐ ┌─────────┐   ┌──────────────┐    │                   │
│         │  │payment-svc│ │notif-svc│   │ search-svc   │    │                   │
│         │  │(Go)       │ │(Go)     │   │ (Go+Elastic) │    │                   │
│         │  │● Stripe   │ │● Email  │   │ ● Product    │    │                   │
│         │  │● Refund   │ │● SMS    │   │   search     │    │                   │
│         │  └─────┬─────┘ └────┬────┘   └──────────────┘    │                   │
│         │        │            │                             │                   │
│         └────────┴────────────┴─────────────────────────────┘                   │
│                               │                                                  │
│  ┌────────────────────────────▼─────────────────────────────────────────────┐   │
│  │                    DATA LAYER                                             │   │
│  │                                                                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐  │   │
│  │  │ Cloud SQL     │  │ Memorystore  │  │    GCS      │  │   Pub/Sub    │  │   │
│  │  │ PostgreSQL    │  │ (Redis)      │  │  (images,   │  │  (async      │  │   │
│  │  │ (users,orders)│  │ (sessions,   │  │   receipts) │  │   events)    │  │   │
│  │  │               │  │  cache)      │  │             │  │              │  │   │
│  │  │ Primary + 2   │  │ 3GB cluster  │  │  Standard   │  │  Topics:     │  │   │
│  │  │ read replicas │  │              │  │  + Nearline  │  │  order-events│  │   │
│  │  └──────────────┘  └──────────────┘  └─────────────┘  │  payment-evts│  │   │
│  │                                                         │  notif-evts  │  │   │
│  │                                                         └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Infrastructure on GKE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GKE CLUSTER: shopfast-prod (us-central1)                  │
│                    Regional cluster — 3 zones for HA                         │
│                                                                              │
│  CONTROL PLANE (Google Managed — no SSH, no cost for regional HA)           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  NODE POOLS                                                          │    │
│  │                                                                      │    │
│  │  ┌──────────────────────────┐   ┌──────────────────────────────┐    │    │
│  │  │  default-pool             │   │  spot-pool                    │    │    │
│  │  │  n2-standard-4 (4vCPU    │   │  n2-standard-4 (Spot VMs)    │    │    │
│  │  │  16GB RAM)               │   │  ~80% cheaper                │    │    │
│  │  │  min: 3, max: 9 nodes    │   │  min: 0, max: 15 nodes       │    │    │
│  │  │  Per zone: 1-3 nodes     │   │  Used for: batch jobs,       │    │    │
│  │  │                          │   │  image processing, reports   │    │    │
│  │  │  Runs: all core services │   │  Toleration required         │    │    │
│  │  └──────────────────────────┘   └──────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  NAMESPACES                                                                  │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  production  │  │   staging     │  │  monitoring  │  │    jenkins   │   │
│  │              │  │               │  │              │  │              │   │
│  │  user-svc    │  │  user-svc     │  │  prometheus  │  │  jenkins-    │   │
│  │  order-svc   │  │  order-svc    │  │  grafana     │  │  controller  │   │
│  │  payment-svc │  │  payment-svc  │  │  alertmanager│  │  jenkins-    │   │
│  │  notif-svc   │  │  (1 replica   │  │  datadog-    │  │  agents      │   │
│  │  inventory   │  │   each)       │  │  agent       │  │  (pods)      │   │
│  │  search-svc  │  │               │  │  dd-cluster- │  │              │   │
│  │  api-gateway │  │               │  │  agent       │  │              │   │
│  └──────────────┘  └───────────────┘  └──────────────┘  └──────────────┘   │
│                                                                              │
│  HPA per service (scales based on CPU + custom RPS metric):                  │
│  order-svc:    min=3, max=30  (critical, highest load)                       │
│  payment-svc:  min=3, max=10  (critical, stateful retries)                   │
│  api-gateway:  min=3, max=20  (frontend of all traffic)                      │
│  user-svc:     min=2, max=15                                                 │
│  notif-svc:    min=2, max=10  (async, tolerable latency)                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Docker — How Each Service is Packaged

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   DOCKER IMAGE BUILD PROCESS (order-svc)                     │
│                                                                              │
│  services/order-svc/                                                         │
│  ├── Dockerfile                                                              │
│  ├── cmd/server/main.go                                                      │
│  ├── internal/                                                               │
│  │   ├── handler/                                                            │
│  │   ├── repository/                                                         │
│  │   └── service/                                                            │
│  ├── go.mod                                                                  │
│  └── go.sum                                                                  │
│                                                                              │
│  # Dockerfile (multi-stage, distroless)                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  FROM golang:1.21-alpine AS builder                                    │  │
│  │    COPY go.mod go.sum ./          ← layer 1: deps (cached most often) │  │
│  │    RUN go mod download            ← layer 2: download (cached)        │  │
│  │    COPY . .                       ← layer 3: source (cache busted)    │  │
│  │    RUN go build -o /order-svc    ← layer 4: compile                  │  │
│  │                                                                        │  │
│  │  FROM gcr.io/distroless/static:nonroot                                │  │
│  │    COPY --from=builder /order-svc /order-svc                          │  │
│  │    USER nonroot:nonroot                                                │  │
│  │    EXPOSE 8082                                                         │  │
│  │    ENTRYPOINT ["/order-svc"]                                           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  Build result: us-central1-docker.pkg.dev/shopfast/services/order-svc:abc123│
│  Image size: ~8 MB (vs 800 MB without multi-stage)                           │
│                                                                              │
│  Each microservice has its own image:                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  ARTIFACT REGISTRY (us-central1-docker.pkg.dev/shopfast/services/)   │   │
│  │                                                                      │   │
│  │  order-svc:abc123def    8 MB    ← current prod                       │   │
│  │  order-svc:bbb456ghi    8 MB    ← previous (kept for rollback)       │   │
│  │  payment-svc:abc123def  9 MB                                         │   │
│  │  user-svc:abc123def     7 MB                                         │   │
│  │  api-gateway:abc123def  6 MB                                         │   │
│  │  notif-svc:abc123def    5 MB                                         │   │
│  │                                                                      │   │
│  │  Vulnerability scanning: automatic on push (CRITICAL → fail build)   │   │
│  │  Retention policy: keep last 10 tags per service                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Git & GitHub — Code Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SHOPFAST REPOSITORY STRUCTURE (GitHub)                    │
│                                                                              │
│  github.com/shopfast/platform (monorepo)                                     │
│  ├── services/                                                               │
│  │   ├── order-svc/                                                          │
│  │   ├── payment-svc/                                                        │
│  │   ├── user-svc/                                                           │
│  │   ├── notif-svc/                                                          │
│  │   └── api-gateway/                                                        │
│  ├── infrastructure/                                                         │
│  │   ├── terraform/          ← GKE, Cloud SQL, VPC, IAM                      │
│  │   └── helm/               ← K8s deployment charts                         │
│  ├── .github/                                                                │
│  │   ├── CODEOWNERS                                                          │
│  │   └── workflows/          ← GitHub Actions (alternative CI)               │
│  └── Jenkinsfile              ← Main CI/CD pipeline                          │
│                                                                              │
│  BRANCH STRATEGY (Trunk-based):                                              │
│                                                                              │
│  main ──●──●──●──────────────●──●──●──────────────────────────►             │
│          │  │                │  │                                            │
│     feature/  feature/  feature/   hotfix/                                   │
│     order-retry  payment-v2  inventory-fix  payment-null-ptr                 │
│     (2h)         (4h)        (1h)           (30min)                          │
│                                                                              │
│  CODEOWNERS rules:                                                           │
│  /services/payment-svc/  → @shopfast/payment-team + @shopfast/sre-team      │
│  /infrastructure/         → @shopfast/sre-team (mandatory review)            │
│  /services/order-svc/    → @shopfast/backend-team                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Jenkins — CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    JENKINS PIPELINE FLOW (on PR / merge to main)             │
│                                                                              │
│  GitHub Event                                                                │
│  (push/PR) ──────────────────────────────────────────────────────────────►  │
│                  │ webhook                                                   │
│                  ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  JENKINS CONTROLLER (namespace: jenkins)                              │   │
│  │  Detects changed service via git diff → spawns K8s pod agent         │   │
│  └──────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
│                                     ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  K8s POD AGENT (ephemeral, namespace: jenkins)                        │   │
│  │  Containers: [jnlp] [golang:1.21] [docker-dind] [kubectl]            │   │
│  │                                                                       │   │
│  │  STAGE 1: DETECT CHANGED SERVICE                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │   │
│  │  │ CHANGED=$(git diff --name-only HEAD~1 | grep "services/" |      │  │   │
│  │  │           cut -d'/' -f2 | sort -u)                              │  │   │
│  │  │ echo "Changed services: order-svc"                              │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  │                  │                                                    │   │
│  │                  ▼                                                    │   │
│  │  STAGE 2: PARALLEL BUILD (only changed services)                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  parallel {                                                   │   │   │
│  │  │    "order-svc" {                                              │   │   │
│  │  │      go test ./...  ← unit tests                             │   │   │
│  │  │      go vet ./...   ← static analysis                        │   │   │
│  │  │      golangci-lint  ← linting                                │   │   │
│  │  │    }                                                          │   │   │
│  │  │  }                                                            │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │     ✓ Pass (< 2 min)     ✗ Fail → PR blocked, Slack alert            │   │
│  │                  │                                                    │   │
│  │                  ▼                                                    │   │
│  │  STAGE 3: DOCKER BUILD + SECURITY SCAN                                │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  docker build -t .../order-svc:$GIT_SHA ./services/order-svc  │   │   │
│  │  │  trivy image --severity HIGH,CRITICAL .../order-svc:$GIT_SHA  │   │   │
│  │  │    CRITICAL CVE found → FAIL (do not push, block PR)          │   │   │
│  │  │    HIGH CVE found     → WARNING (allow, create Jira ticket)   │   │   │
│  │  │  docker push .../order-svc:$GIT_SHA                           │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                  │                                                    │   │
│  │  [PR builds stop here — image pushed but not deployed]                │   │
│  │                  │                                                    │   │
│  │  STAGE 4: DEPLOY TO STAGING  (main branch only)                       │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  helm upgrade order-svc ./infrastructure/helm/order-svc       │   │   │
│  │  │    --namespace staging                                         │   │   │
│  │  │    --set image.tag=$GIT_SHA                                    │   │   │
│  │  │    -f ./infrastructure/helm/order-svc/values.staging.yaml      │   │   │
│  │  │  kubectl rollout status deployment/order-svc -n staging        │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                  │                                                    │   │
│  │  STAGE 5: INTEGRATION TESTS                                           │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  ./scripts/integration-test.sh https://staging.shopfast.io    │   │   │
│  │  │  Tests:                                                        │   │   │
│  │  │  ✓ POST /api/orders → 201, order ID returned                  │   │   │
│  │  │  ✓ GET /api/orders/:id → 200, correct payload                 │   │   │
│  │  │  ✓ POST /api/orders/:id/cancel → 200                          │   │   │
│  │  │  ✓ Payment flow end-to-end (Stripe test mode)                 │   │   │
│  │  │  ✓ P99 latency < 500ms under 50 concurrent users              │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                  │                                                    │   │
│  │  STAGE 6: PROD DEPLOY (manual gate for payment-svc, auto for others)  │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  input {                                                       │   │   │
│  │  │    message: "Deploy order-svc $GIT_SHA to production?"        │   │   │
│  │  │    submitter: "sre-team,senior-devs"                          │   │   │
│  │  │    timeout: 2h                                                 │   │   │
│  │  │  }                                                             │   │   │
│  │  │  helm upgrade order-svc ... --namespace production             │   │   │
│  │  │    --set image.tag=$GIT_SHA                                    │   │   │
│  │  │    --set replicaCount=5                                        │   │   │
│  │  │    -f values.production.yaml                                   │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                  │                                                    │   │
│  │  STAGE 7: POST-DEPLOY VALIDATION                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │  Wait 5 minutes, check:                                        │   │   │
│  │  │  ● Error rate < 0.1% (query Prometheus)                       │   │   │
│  │  │  ● P99 < 500ms (query Prometheus)                             │   │   │
│  │  │  ● Pod restarts = 0 (kubectl)                                 │   │   │
│  │  │  ● Datadog synthetic check: /api/orders/health returns 200    │   │   │
│  │  │                                                                │   │   │
│  │  │  FAIL → automatic rollback:                                    │   │   │
│  │  │    kubectl rollout undo deployment/order-svc -n production     │   │   │
│  │  │    Slack: "🔴 Auto-rollback triggered for order-svc"          │   │   │
│  │  │  PASS → Slack: "✅ order-svc deployed to prod"                │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Kubernetes — How Services Run in Production

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              order-svc DEPLOYMENT (production namespace)                      │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Deployment: order-svc                                                  │ │
│  │  replicas: 5  strategy: RollingUpdate (maxUnavailable=0, maxSurge=2)   │ │
│  │                                                                         │ │
│  │  Pod Spec:                                                              │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │  containers:                                                     │  │ │
│  │  │  - name: order-svc                                               │  │ │
│  │  │    image: .../order-svc:abc123def                                │  │ │
│  │  │    ports: [8082]                                                 │  │ │
│  │  │    resources:                                                    │  │ │
│  │  │      requests: {cpu: 200m, memory: 256Mi}  ← scheduling         │  │ │
│  │  │      limits:   {cpu: 1000m, memory: 512Mi} ← hard cap           │  │ │
│  │  │    env:                                                          │  │ │
│  │  │    - DB_HOST: (from ConfigMap)                                   │  │ │
│  │  │    - DB_PASS: (from Secret → GCP Secret Manager)                │  │ │
│  │  │    - DD_SERVICE: order-svc                                       │  │ │
│  │  │    - DD_ENV: production                                          │  │ │
│  │  │    - DD_VERSION: abc123def                                       │  │ │
│  │  │    readinessProbe:                                               │  │ │
│  │  │      httpGet: {path: /ready, port: 8082}                        │  │ │
│  │  │      initialDelaySeconds: 10                                     │  │ │
│  │  │      periodSeconds: 5                                            │  │ │
│  │  │    livenessProbe:                                                │  │ │
│  │  │      httpGet: {path: /healthz, port: 8082}                      │  │ │
│  │  │      initialDelaySeconds: 30                                     │  │ │
│  │  │      failureThreshold: 3                                         │  │ │
│  │  │  - name: cloudsql-proxy             ← sidecar for DB access     │  │ │
│  │  │    image: cloud-sql-proxy:2.0                                    │  │ │
│  │  │    args: [shopfast:us-central1:prod-db]                          │  │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  HPA: order-svc                                                      │   │
│  │  minReplicas: 3   maxReplicas: 30                                    │   │
│  │  metrics:                                                            │   │
│  │  - cpu utilization target: 70%                                       │   │
│  │  - custom: orders_per_second > 100 per pod                          │   │
│  │                                                                      │   │
│  │  NORMAL:   5 pods  (200 orders/min)                                  │   │
│  │  PEAK:     18 pods (500 orders/min) ← Black Friday                   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  PodDisruptionBudget: order-svc                                      │   │
│  │  minAvailable: 3    ← never go below 3 running pods (maintenance)    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  SERVICE MESH (pod-to-pod communication):                                    │
│                                                                              │
│  api-gateway-pod                                                             │
│       │  HTTP POST /api/v1/orders                                            │
│       │  Host: order-svc.production.svc.cluster.local:8082                  │
│       ▼                                                                      │
│  order-svc Service (ClusterIP)                                               │
│       │  kube-proxy routes to one of 5 pods (iptables round-robin)          │
│       ▼                                                                      │
│  order-svc Pod (one of 5)                                                    │
│       │  POST /api/v1/payment → payment-svc.production.svc.cluster.local    │
│       │  POST /api/v1/reserve → inventory-svc.production.svc.cluster.local  │
│       │  PUBLISH order.created → Pub/Sub                                     │
│       ▼                                                                      │
│  [response back up the chain]                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Monitoring Stack — Full Observability

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MONITORING ARCHITECTURE                                    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  PROMETHEUS (namespace: monitoring)                                    │  │
│  │                                                                       │  │
│  │  Scrape targets (every 30s):                                          │  │
│  │  ● order-svc pods         → /metrics (custom + Go runtime metrics)   │  │
│  │  ● payment-svc pods       → /metrics                                 │  │
│  │  ● kube-state-metrics     → K8s object state (pod status, restarts)  │  │
│  │  ● node-exporter          → host CPU, memory, disk, network          │  │
│  │  ● cloud-sql-proxy        → DB connection pool metrics               │  │
│  │  ● redis-exporter         → Memorystore hits/misses, memory          │  │
│  │                                                                       │  │
│  │  Custom metrics exposed by order-svc:                                 │  │
│  │  ● orders_created_total{status="paid|failed|pending"}                │  │
│  │  ● order_processing_duration_seconds{step="payment|inventory"}       │  │
│  │  ● orders_per_second (gauge, updated every 10s)                      │  │
│  │  ● payment_retries_total{reason="timeout|declined|error"}            │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                    │                                                          │
│                    ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  ALERT RULES (PrometheusRule CRD)                                     │  │
│  │                                                                       │  │
│  │  group: shopfast.critical                                             │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │  PaymentHighFailureRate:                                        │  │  │
│  │  │    expr: rate(orders_created_total{status="failed"}[5m]) /      │  │  │
│  │  │          rate(orders_created_total[5m]) > 0.02                  │  │  │
│  │  │    for: 3m                                                      │  │  │
│  │  │    severity: critical                                           │  │  │
│  │  │    → PagerDuty page (wake on-call)                              │  │  │
│  │  │                                                                 │  │  │
│  │  │  OrderSvcHighLatency:                                           │  │  │
│  │  │    expr: histogram_quantile(0.99,                               │  │  │
│  │  │           rate(order_processing_duration_seconds_bucket[5m])    │  │  │
│  │  │          ) > 2.0                                                │  │  │
│  │  │    for: 5m   severity: warning                                  │  │  │
│  │  │    → Slack #sre-alerts                                          │  │  │
│  │  │                                                                 │  │  │
│  │  │  HPAAtMaxReplicas:                                              │  │  │
│  │  │    expr: kube_hpa_status_current_replicas ==                    │  │  │
│  │  │          kube_hpa_spec_max_replicas                             │  │  │
│  │  │    for: 10m  severity: warning                                  │  │  │
│  │  │    → Slack (may need to increase maxReplicas)                   │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                    │                                                          │
│                    ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  GRAFANA DASHBOARDS                                                   │  │
│  │                                                                       │  │
│  │  Dashboard 1: ShopFast Operations (RED method)                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │  [Orders/sec]  [Error Rate %]  [P99 Latency]  [Active Pods]     │ │  │
│  │  │  Graph: orders by status (paid/failed/pending) over time        │ │  │
│  │  │  Graph: payment failure rate with SLO line (2%)                 │ │  │
│  │  │  Graph: order processing P50/P95/P99 over 24h                  │ │  │
│  │  │  Table: top 5 error types in last 1h                            │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  │  Dashboard 2: Infrastructure (USE method)                             │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │  [Node CPU%]  [Node Mem%]  [Pod Restarts]  [Disk I/O]           │ │  │
│  │  │  Heatmap: pod CPU usage by service                              │ │  │
│  │  │  HPA status: current vs max replicas per service                │ │  │
│  │  │  DB connections: current/max per Cloud SQL instance             │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  │  Dashboard 3: SLO Dashboard                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Payment Success Rate SLO: 99.9%                                │ │  │
│  │  │  Error budget remaining: 32.4 min (of 43.8 min/month)          │ │  │
│  │  │  Burn rate: 0.8x (healthy — under 1x)                          │ │  │
│  │  │  Events overlay: deployments, incidents                         │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  DATADOG (APM + Logs + Synthetic)                                     │  │
│  │                                                                       │  │
│  │  APM Service Map:                                                     │  │
│  │  api-gateway ──► order-svc ──► payment-svc ──► Stripe               │  │
│  │                     │                                                │  │
│  │                     ├──────► inventory-svc ──► Cloud SQL             │  │
│  │                     └──────► Pub/Sub ──────► notif-svc ──► SendGrid  │  │
│  │                                                                       │  │
│  │  Each hop shows: avg latency, P99, error rate, requests/sec          │  │
│  │                                                                       │  │
│  │  Synthetic monitors (run every 5 min from 3 regions):                │  │
│  │  ● POST /api/orders (checkout flow) → assert 201 + order ID         │  │
│  │  ● GET /api/orders/:id → assert 200 + correct status                │  │
│  │  ● Homepage load time < 2s                                           │  │
│  │  Alert if any fails from 2+ regions: page on-call immediately        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. GCP Services — How They're Used

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GCP SERVICES IN SHOPFAST                                  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  COMPUTE                                                                │ │
│  │  GKE (shopfast-prod, us-central1, regional)                            │ │
│  │  └─ All microservices, monitoring, Jenkins agents                      │ │
│  │                                                                        │ │
│  │  Cloud Run (admin tools, one-off jobs — no need to manage K8s for     │ │
│  │             low-traffic internal services)                             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  STORAGE                                                                │ │
│  │  Cloud SQL (PostgreSQL 15, us-central1)                                │ │
│  │  ├─ prod-users-db:  user accounts, auth tokens                        │ │
│  │  ├─ prod-orders-db: orders, order items, payment records               │ │
│  │  └─ Both: REGIONAL HA, daily backups, 7-day PITR, private IP only     │ │
│  │                                                                        │ │
│  │  Memorystore (Redis 7, 3GB)                                            │ │
│  │  ├─ Session cache: user JWT tokens (TTL: 24h)                         │ │
│  │  ├─ Product cache: inventory counts (TTL: 60s)                        │ │
│  │  └─ Rate limiting: per-user request counts                            │ │
│  │                                                                        │ │
│  │  GCS (Cloud Storage)                                                   │ │
│  │  ├─ gs://shopfast-product-images/  (public, CDN-backed)               │ │
│  │  ├─ gs://shopfast-receipts/        (private, customer PDFs)           │ │
│  │  ├─ gs://shopfast-tf-state/        (Terraform state, versioned)       │ │
│  │  └─ gs://shopfast-db-backups/      (Cloud SQL exports, Nearline)      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  MESSAGING                                                              │ │
│  │  Pub/Sub topics:                                                       │ │
│  │  ├─ order-events    → notif-svc (email), analytics, fraud-detector    │ │
│  │  ├─ payment-events  → accounting-svc, audit-log-svc                   │ │
│  │  └─ inventory-events → search-svc (re-index), analytics               │ │
│  │                                                                        │ │
│  │  Flow: order-svc creates order → publishes to order-events →          │ │
│  │        notif-svc picks up → sends confirmation email via SendGrid     │ │
│  │        (fully async, order-svc doesn't wait for email)                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  SECURITY                                                               │ │
│  │  Secret Manager:                                                       │ │
│  │  ├─ stripe-api-key         → payment-svc (via Workload Identity)      │ │
│  │  ├─ db-password-orders     → order-svc                                │ │
│  │  ├─ jwt-signing-key        → user-svc                                 │ │
│  │  └─ sendgrid-api-key       → notif-svc                                │ │
│  │  Each secret: auto-rotation every 90 days, audit logged               │ │
│  │                                                                        │ │
│  │  Workload Identity:                                                    │ │
│  │  ├─ order-svc KSA    → order-svc-sa@shopfast.iam.gsa.com             │ │
│  │  │    permissions: cloudsql.client, pubsub.publisher, secretAccessor  │ │
│  │  └─ No JSON keys anywhere! IAM-based pod authentication               │ │
│  │                                                                        │ │
│  │  Cloud Armor:                                                          │ │
│  │  ├─ SQLi/XSS OWASP rules                                              │ │
│  │  ├─ Rate limit: 1000 req/min per IP                                   │ │
│  │  └─ Geo-block: block TOR exit nodes                                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CI/CD                                                                  │ │
│  │  Artifact Registry: us-central1-docker.pkg.dev/shopfast/services/      │ │
│  │  ├─ Stores all service Docker images                                   │ │
│  │  ├─ Vulnerability scanning on every push                               │ │
│  │  └─ Binary Authorization: only signed images can deploy to prod        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Complete Request Flow — One Order, Every Tool Involved

```
┌─────────────────────────────────────────────────────────────────────────────┐
│          TRACE: User places an order (every tool's role shown)               │
│                                                                              │
│  USER'S BROWSER                                                              │
│  POST https://shopfast.io/api/orders                                         │
│  Body: {items: [{product_id: "p123", qty: 2}], payment_method: "card"}      │
│    │                                                                         │
│    │ DNS → Cloud DNS → 34.x.x.x (Global Load Balancer anycast IP)          │
│    ▼                                                                         │
│  CLOUD ARMOR                                                                 │
│    ● Checks: is this IP rate-limited? Any SQLi patterns? → PASS             │
│    │                                                                         │
│    ▼                                                                         │
│  HTTPS LOAD BALANCER (GCP)                                                   │
│    ● SSL termination (Google-managed cert)                                   │
│    ● Route /api/* → api-gateway backend                                      │
│    │                                                                         │
│    ▼                                                                         │
│  API GATEWAY POD (GKE, production namespace)                                 │
│    ● Validate JWT token → call user-svc to verify                           │
│    ● Rate limit check (Redis: user_id:rate_limit)                           │
│    ● Route to order-svc                                                      │
│    ● Datadog APM: root span starts here, trace_id = abc123                  │
│    │                                                                         │
│    │ Service DNS: order-svc.production.svc.cluster.local:8082               │
│    ▼                                                                         │
│  ORDER-SVC POD (GKE)                                                         │
│    ● Emit metric: orders_created_total.Inc()                                │
│    ● Log: {"trace_id":"abc123","user_id":"u456","msg":"order started"}      │
│    ●                                                                         │
│    ├─► INVENTORY-SVC: reserve 2x p123 (gRPC)                                │
│    │     ● Inventory-svc checks Redis cache first (60s TTL)                 │
│    │     ● Cache miss → query Cloud SQL (prod-orders-db via Cloud SQL proxy)│
│    │     ● Reserve stock → UPDATE inventory SET reserved=reserved+2         │
│    │     ● Return: "reserved" ✓                                             │
│    │                                                                         │
│    ├─► PAYMENT-SVC: charge card (HTTP)                                       │
│    │     ● Fetch Stripe API key from Secret Manager (Workload Identity)     │
│    │     ● Stripe API call: POST /v1/payment_intents                        │
│    │     ● Retry logic: 3 attempts, exp backoff if timeout                  │
│    │     ● Return: {status: "succeeded", charge_id: "ch_abc"}               │
│    │                                                                         │
│    ├─► CLOUD SQL (via proxy sidecar):                                        │
│    │     INSERT INTO orders (user_id, items, total, status)                 │
│    │     VALUES ('u456', '...', 59.99, 'paid')                              │
│    │     → order_id: "ord_789"                                              │
│    │                                                                         │
│    └─► PUB/SUB: publish order-events                                         │
│          {order_id: "ord_789", user_id: "u456", status: "paid", ...}        │
│          (async — order-svc doesn't wait for downstream)                    │
│    │                                                                         │
│    ● Emit metric: order_processing_duration_seconds.Observe(0.045)          │
│    │                                                                         │
│    ▼                                                                         │
│  RESPONSE: 201 Created                                                       │
│  {order_id: "ord_789", status: "paid", estimated_delivery: "2026-04-02"}    │
│    │                                                                         │
│    │ Total latency: 45ms  ← P99 target is 500ms, we're well within          │
│    ▼                                                                         │
│  USER SEES: "Order confirmed!" ✓                                             │
│                                                                              │
│  ASYNC (Pub/Sub → notif-svc):                                                │
│    notif-svc receives order-events message                                   │
│    → fetch user email from user-svc                                          │
│    → send confirmation email via SendGrid API                                │
│    → store receipt PDF in GCS: gs://shopfast-receipts/ord_789.pdf           │
│    → ack Pub/Sub message                                                     │
│                                                                              │
│  OBSERVABILITY (happening in parallel throughout):                           │
│    Prometheus: scrapes /metrics from all pods every 30s                      │
│    Datadog APM: trace abc123 stored, shows full breakdown:                   │
│      api-gateway[2ms] → order-svc[45ms]                                     │
│        ├─ inventory-svc[5ms]                                                 │
│        ├─ payment-svc[35ms] → Stripe[30ms]                                  │
│        └─ Cloud SQL[3ms]                                                     │
│    Cloud Logging: all JSON logs indexed, searchable by trace_id              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Incident: Payment Spike on Black Friday

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              REAL INCIDENT WALKTHROUGH — Black Friday 2025                   │
│                                                                              │
│  CONTEXT: 18:00 UTC, Black Friday peak. Traffic 3x normal.                  │
│                                                                              │
│  18:04 — Datadog synthetic monitor FAILS                                    │
│           "POST /api/orders returned 503 from us-east1 region"              │
│    │                                                                         │
│    ▼                                                                         │
│  18:04 — PagerDuty pages on-call SRE (Chethan)                             │
│           "CRITICAL: PaymentHighFailureRate = 8.4% (threshold: 2%)"         │
│    │                                                                         │
│    ▼                                                                         │
│  18:06 — SRE opens Datadog dashboard                                        │
│           Sees: payment-svc P99 latency = 12s (normal: 200ms)               │
│           Deployment overlay: no recent deploy                               │
│           HPA status: payment-svc at 10/10 replicas (MAX!)                 │
│    │                                                                         │
│    ▼                                                                         │
│  18:07 — Check Grafana: HPAAtMaxReplicas fired 8 minutes ago (warning)      │
│           (Warning was not paged, only Slack — on-call missed it)           │
│           Root cause hypothesis: payment-svc saturated, can't scale further │
│    │                                                                         │
│    ▼                                                                         │
│  18:08 — IMMEDIATE MITIGATION:                                              │
│           kubectl patch hpa payment-svc -n production                       │
│             -p '{"spec":{"maxReplicas":25}}'                                 │
│           K8s Cluster Autoscaler: detects pending pods, adds 3 new nodes    │
│           (takes ~3 min to provision GCE nodes)                             │
│    │                                                                         │
│    ▼                                                                         │
│  18:09 — Check Datadog APM trace of failed payment:                         │
│           payment-svc → Stripe API: timeout after 10s                       │
│           Stripe status page: "Elevated latency in us-central1"             │
│           REAL root cause: Stripe API slow, payment-svc not failing fast    │
│    │                                                                         │
│    ▼                                                                         │
│  18:11 — SECOND MITIGATION:                                                  │
│           payment-svc has 10s timeout for Stripe. Reduce to 3s.             │
│           kubectl set env deployment/payment-svc                            │
│             STRIPE_TIMEOUT_MS=3000 -n production                            │
│           kubectl rollout status deployment/payment-svc -n production       │
│    │                                                                         │
│    ▼                                                                         │
│  18:14 — New nodes ready, more pods scheduled                               │
│           Error rate drops: 8.4% → 3.2% → 1.1% → 0.4%                    │
│           PagerDuty alert resolves                                           │
│    │                                                                         │
│    ▼                                                                         │
│  18:20 — Declare incident resolved                                           │
│           Post to Slack: "payment-svc incident resolved, 16 min duration"   │
│           Error budget consumed: 16 min of 43.8 min/month = 36%            │
│           Remaining error budget: ~28 min                                   │
│    │                                                                         │
│    ▼                                                                         │
│  POSTMORTEM (next day):                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Root cause: Stripe API elevated latency (external) + our timeout    │   │
│  │  was too high (10s). Under high concurrency, slow Stripe calls       │   │
│  │  occupied all goroutines → payment-svc saturated.                    │   │
│  │                                                                       │   │
│  │  Contributing factors:                                                │   │
│  │  - HPAAtMaxReplicas was WARNING (not paged) — missed the signal      │   │
│  │  - maxReplicas was set too low (10) for Black Friday scale           │   │
│  │  - No circuit breaker for Stripe calls                               │   │
│  │                                                                       │   │
│  │  Action items:                                                        │   │
│  │  1. Reduce Stripe timeout to 3s permanently           (P1, done)    │   │
│  │  2. Add circuit breaker for payment provider calls    (P1, 1 week)  │   │
│  │  3. Pre-scale payment-svc before Black Friday         (P1, process) │   │
│  │  4. Upgrade HPAAtMaxReplicas to CRITICAL alert        (P2, 2 days)  │   │
│  │  5. Add Stripe status page to runbook                 (P2, 1 day)   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Everything Together — Master Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        SHOPFAST — ALL TOOLS INTERACTION MAP                          │
│                                                                                      │
│   DEVELOPER                    SOURCE CONTROL              CI/CD                     │
│   ┌──────────┐                 ┌──────────────┐           ┌───────────────────────┐  │
│   │  VSCode  │──git push──────►│    GitHub    │──webhook─►│       Jenkins         │  │
│   │  + tests │                 │              │           │  K8s pod agents        │  │
│   │  + docker│                 │  Monorepo    │           │  Jenkinsfile (shared  │  │
│   │  compose │                 │  CODEOWNERS  │           │  library)             │  │
│   └──────────┘                 │  Branch prot.│           │  Stages:              │  │
│        │                       │  PR review   │           │  test→build→scan→     │  │
│        │ docker-compose up      └──────────────┘           │  push→stage→int-test │  │
│        │ (local dev)                                       │  →gate→prod→validate │  │
│        ▼                                                   └──────────┬────────────┘  │
│   ┌────────────────────┐                                              │               │
│   │  Local services    │                                    docker push               │
│   │  + postgres        │                                              │               │
│   │  + redis           │                                              ▼               │
│   └────────────────────┘                                   ┌──────────────────────┐  │
│                                                            │  Artifact Registry   │  │
│                                                            │  (GCR)               │  │
│                                                            │  Images per service  │  │
│                                                            │  CVE scan per image  │  │
│                                                            └──────────┬───────────┘  │
│                                                                       │ image pull    │
│                                                                       ▼               │
│   KUBERNETES (GKE)                                                                   │
│   ┌───────────────────────────────────────────────────────────────────────────────┐  │
│   │  production namespace                  monitoring namespace                    │  │
│   │  ┌────────────────────────────┐        ┌────────────────────────────────────┐ │  │
│   │  │  order-svc (5 pods, HPA)   │        │  Prometheus (scrapes /metrics)     │ │  │
│   │  │  payment-svc (5 pods, HPA) │◄──────►│  Grafana (dashboards)              │ │  │
│   │  │  user-svc (3 pods)         │        │  Alertmanager → PagerDuty/Slack    │ │  │
│   │  │  api-gateway (3 pods)      │        │  Datadog Agent (DaemonSet)         │ │  │
│   │  │  notif-svc (2 pods)        │        │  Datadog Cluster Agent             │ │  │
│   │  │  inventory-svc (3 pods)    │        └────────────────────────────────────┘ │  │
│   │  └─────────────┬──────────────┘                                                │  │
│   │                │                                                                │  │
│   │         Workload Identity: pods auth to GCP APIs without keys                  │  │
│   └───────────────────────────────────────────────────────────────────────────────┘  │
│                    │                                                                   │
│                    ▼                                                                   │
│   GCP SERVICES                                                                        │
│   ┌───────────────────────────────────────────────────────────────────────────────┐  │
│   │  Cloud SQL      Memorystore     GCS           Pub/Sub        Secret Manager  │  │
│   │  (PostgreSQL)   (Redis)         (objects)     (async events) (credentials)   │  │
│   │                                                                               │  │
│   │  Cloud Armor    HTTPS LB        Cloud DNS     Cloud Logging  Cloud Monitoring│  │
│   │  (WAF/DDoS)     (global, CDN)   (DNS)         (log sink)     (GCP metrics)   │  │
│   └───────────────────────────────────────────────────────────────────────────────┘  │
│                    │                                                                   │
│                    ▼                                                                   │
│   OBSERVABILITY (Datadog Cloud)                                                        │
│   ┌───────────────────────────────────────────────────────────────────────────────┐  │
│   │  APM: service map, traces, latency by service+endpoint                        │  │
│   │  Logs: all container logs, parsed, searchable, correlated with traces         │  │
│   │  Metrics: infra + custom business metrics                                     │  │
│   │  Synthetics: 5-min checks from 3 regions                                     │  │
│   │  SLOs: payment success 99.9%, order create P99<500ms                         │  │
│   │  Monitors → PagerDuty → On-call SRE → Runbook → Resolve → Postmortem        │  │
│   └───────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘

TOOL RESPONSIBILITY SUMMARY:
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Git/GitHub   → Source truth, code review, branch protection, changelog             │
│  Docker        → Package app as immutable artifact, multi-stage, distroless         │
│  Artifact Reg  → Store + scan images, immutable tags, supply chain security         │
│  Jenkins       → Automate test+build+scan+deploy, gates, notifications              │
│  Kubernetes    → Run containers, self-heal, scale, rolling updates, service mesh    │
│  GKE           → Managed K8s, node autoscale, Workload Identity, GCP integration   │
│  Prometheus    → Collect metrics, evaluate alert rules, custom app metrics          │
│  Alertmanager  → Route+deduplicate+silence alerts, PagerDuty/Slack integration      │
│  Grafana       → Business + infra dashboards, SLO panels, event overlays            │
│  Datadog       → APM traces, correlated logs, synthetic monitoring, SLO tracking    │
│  Cloud SQL     → Managed PostgreSQL with HA, PITR, private networking               │
│  Memorystore   → Managed Redis for cache, sessions, rate limiting                  │
│  Pub/Sub       → Async event bus, decouple order→notification→analytics             │
│  Secret Mgr    → Centralized secrets, rotation, audit trail, no JSON keys           │
│  Cloud Armor   → WAF + DDoS protection, rate limiting, OWASP rules                  │
│  GCS           → Object storage for images, receipts, backups, Terraform state      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. How to Answer "Tell Me About a Project" in Interview

Use ShopFast as your example. Structure:

```
SITUATION:
  "I worked on a microservices e-commerce platform serving 50k DAU,
   processing 500 orders/minute at peak. We had 8 engineers and a
   2-person SRE team responsible for reliability and deployments."

ARCHITECTURE (draw on whiteboard):
  "We ran 6 Go microservices on GKE. The order service was the core —
   it coordinated inventory reservation, payment via Stripe, and async
   notifications through Pub/Sub. Each service had its own Docker image
   built with multi-stage Dockerfiles, stored in Artifact Registry."

CI/CD:
  "We used Jenkins with Kubernetes pod agents — every PR spawned a
   fresh pod, ran tests, built the Docker image, ran Trivy CVE scan.
   On merge to main, it deployed to staging, ran integration tests,
   then required SRE approval for payment-svc production deploys.
   Post-deploy, we validated error rate and P99 against Prometheus
   and auto-rolled back if thresholds were breached."

MONITORING:
  "We had two monitoring layers: Prometheus+Grafana for infrastructure
   and custom business metrics (orders/sec, payment failure rate), and
   Datadog for APM and correlated logs. This meant when an alert fired,
   we could go from the metric spike → the slow trace → the exact log
   line in under 2 minutes."

INCIDENT EXAMPLE (Black Friday):
  "Our most memorable incident was Black Friday — Stripe had elevated
   latency, our 10-second timeout was too generous, and payment-svc
   goroutines backed up. We were at maxReplicas (10) and couldn't
   scale further. I patched the HPA maxReplicas to 25 live, reduced
   the Stripe timeout to 3 seconds, and cluster autoscaler added nodes
   within 3 minutes. We went from 8% error rate back to <0.4% in
   about 16 minutes. Postmortem action item was adding a circuit
   breaker for external payment calls — which we shipped the next week."

SRE OUTCOME:
  "We maintained 99.94% monthly availability against a 99.9% SLO.
   By automating our toil (deployment gates, auto-rollback, runbooks),
   we reduced on-call burden from 8 pages/week to under 2."
```
