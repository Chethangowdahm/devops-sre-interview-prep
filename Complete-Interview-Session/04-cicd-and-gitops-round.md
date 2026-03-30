# Complete Interview Session — File 4: CI/CD & GitOps Round

> **Round Type:** Deep dive into CI/CD stack — Jenkins, ArgoCD, GitHub Actions
> **Your Edge:** Live GitOps at Vendasta using ArgoCD + GitHub Actions on GKE.

---

## SECTION 1: JENKINS

---

### Q1. Declarative vs Scripted Pipelines in Jenkins?

**[YOUR ANSWER]:**

**Declarative Pipeline:**
- Structured, opinionated syntax using pipeline block
- Built-in stages, steps, post conditions
- Easier to read, validate, and maintain
- Has syntax validation before execution — Recommended for most use cases

**Scripted Pipeline:**
- Pure Groovy code inside node block
- More flexible — use loops, conditionals anywhere
- Harder to read, no built-in structure

**Declarative Example:**
```groovy
pipeline {
  agent { label 'k8s-agent' }
  stages {
    stage('Build') { steps { sh 'docker build -t myimage .' } }
    stage('Test') { steps { sh 'go test ./...' } }
    stage('Push') { steps { sh 'docker push myimage' } }
  }
  post {
    success { slackSend message: 'Build passed!' }
    failure { slackSend message: 'Build FAILED!' }
  }
}
```

**At Vendasta:** All our pipelines are declarative. I maintain a shared library of reusable steps.

---

### Q2. How do you run Jenkins pipelines on Kubernetes?

**[YOUR ANSWER]:**

Using the Jenkins Kubernetes plugin — each build runs as an ephemeral pod, destroyed when done.

```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: go
            image: golang:1.21
            command: [sleep, infinity]
      '''
    }
  }
  stages {
    stage('Build') {
      steps { container('go') { sh 'go build ./...' } }
    }
  }
}
```

**Benefits:** No idle agents, isolated builds, elastic scaling, cost-efficient.
**At Vendasta:** Build concurrency went from 5 to 40+ without any capacity planning.

---

### Q3. How do you handle secrets in Jenkins pipelines?

**[YOUR ANSWER]:**

Never hardcode secrets. Use Jenkins credentials binding:

```groovy
pipeline {
  environment {
    DOCKER_CREDS = credentials('docker-registry-secret')
    API_TOKEN = credentials('api-token-secret')
  }
  stages {
    stage('Push') {
      steps {
        sh 'echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin'
      }
    }
  }
}
```

**At Vendasta:** Jenkins credentials store for build secrets, GCP Workload Identity for cloud access — no static keys ever.

---

## SECTION 2: ARGOCD & GITOPS

---

### Q4. What is GitOps and how is it different from traditional CI/CD?

**[YOUR ANSWER]:**

GitOps is an operational framework where Git is the single source of truth for both infrastructure and application state.

**Traditional CI/CD (Push model):**
- Pipeline pushes changes to cluster
- State can drift from definition
- Manual interventions create undocumented changes

**GitOps (Pull model):**
- Git repo defines desired state
- ArgoCD continuously watches Git and pulls changes
- Manual changes are automatically reverted — drift correction
- Full audit log via Git history
- Rollback = git revert

**At Vendasta:** Migrated from Jenkins push deployments to GitOps with ArgoCD. Every production change is a PR with review — full auditability.

---

### Q5. Walk me through ArgoCD architecture and sync strategies.

**[YOUR ANSWER]:**

ArgoCD is a declarative GitOps CD tool for Kubernetes.

**How it works:**
1. Define Application resource pointing to Git repo + target cluster/namespace
2. ArgoCD polls repo every 3 min or via webhook
3. Compares live cluster state vs desired Git state
4. If OutOfSync, applies the diff

**Application YAML:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api
spec:
  source:
    repoURL: https://github.com/vendasta/k8s-manifests
    targetRevision: main
    path: services/payment-api
    helm:
      valueFiles: [values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Sync Strategies:**
- Manual sync: Review diff in UI, click sync — for production safety
- Auto sync: Syncs automatically when Git changes — for dev/staging
- selfHeal: Reverts any manual kubectl changes to match Git
- prune: Deletes resources removed from Git

**At Vendasta:** Production = manual sync with Slack alert. Staging = fully auto-sync.

---

### Q6. What are ArgoCD ApplicationSets?

**[YOUR ANSWER]:**

ApplicationSets generate multiple ArgoCD Applications from a single template — great for managing many microservices.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
spec:
  generators:
  - list:
      elements:
      - service: payment-api
        env: production
      - service: user-api
        env: production
      - service: notification-api
        env: production
  template:
    metadata:
      name: '{{service}}-{{env}}'
    spec:
      source:
        repoURL: https://github.com/vendasta/k8s-manifests
        path: 'services/{{service}}'
        helm:
          valueFiles: ['values-{{env}}.yaml']
      destination:
        namespace: '{{env}}'
```

**At Vendasta:** I manage 15+ microservices with one ApplicationSet — adding a service is just one line in the elements list.

---

## SECTION 3: GITHUB ACTIONS

---

### Q7. How do you implement GitOps CI/CD with GitHub Actions?

**[YOUR ANSWER]:**

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true
    - run: go test ./... -race

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
    - uses: actions/checkout@v4
    - uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
        service_account: github-actions@vendasta-prod.iam.gserviceaccount.com
    - run: |
        gcloud auth configure-docker gcr.io
        docker build -t gcr.io/vendasta-prod/payment-api:$GITHUB_SHA .
        docker push gcr.io/vendasta-prod/payment-api:$GITHUB_SHA

  update-manifest:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        repository: vendasta/k8s-manifests
        token: ${{ secrets.MANIFEST_PAT }}
    - run: |
        sed -i 's|payment-api:.*|payment-api:${{ github.sha }}|' services/payment-api/values-prod.yaml
        git commit -am 'chore: update payment-api'
        git push
```

**At Vendasta:** CI builds and pushes image, then updates manifest repo — ArgoCD detects and deploys. No pipeline needs cluster credentials.

---

### Q8. What is OIDC authentication in GitHub Actions and why is it better?

**[YOUR ANSWER]:**

OIDC lets GitHub Actions authenticate with cloud providers without storing long-lived credentials.

**Traditional (bad):** Store GCP JSON key as GitHub Secret — never expires, high risk if GitHub is breached.

**OIDC (good):** GitHub issues a short-lived token per job. GCP validates and exchanges for a short-lived access token. No secrets stored anywhere.

```yaml
permissions:
  id-token: write
steps:
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/NUM/locations/global/workloadIdentityPools/POOL/providers/PROVIDER'
    service_account: 'github-actions@PROJECT.iam.gserviceaccount.com'
```

**At Vendasta:** Migrated all GitHub Actions to OIDC — eliminated secret rotation and passed our security audit.

---

### Q9. Explain canary deployment strategy with ArgoCD Rollouts.

**[YOUR ANSWER]:**

```yaml
strategy:
  canary:
    steps:
    - setWeight: 10
    - pause: {duration: 5m}
    - setWeight: 50
    - pause: {duration: 10m}
    - setWeight: 100
    analysis:
      templates:
      - templateName: success-rate
      args:
      - name: service-name
        value: payment-api
```

The AnalysisTemplate queries Datadog/Prometheus — if error rate > 1%, rollout is automatically halted and rolled back.

**At Vendasta:** Implemented after an OOMKill incident. Caught 2 bad deployments before full rollout, saving ~15 min of production impact each time.

---

*Next: Move to File 05 — Cloud AWS & GCP Round*
