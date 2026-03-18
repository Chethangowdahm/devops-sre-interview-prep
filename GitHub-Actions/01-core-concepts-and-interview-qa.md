# GitHub Actions - Core Concepts & Interview Q&A
> Senior DevOps/SRE Level | Modern CI/CD

---

## CORE CONCEPTS

### What is GitHub Actions?
GitHub-native CI/CD platform. Event-driven automation defined as YAML workflows in .github/workflows/.

### Key Components
| Component | Description |
|---|---|
| **Workflow** | Automated process triggered by events. YAML file in .github/workflows/ |
| **Event** | Trigger: push, pull_request, schedule, workflow_dispatch, etc. |
| **Job** | Set of steps running on a runner. Jobs run in parallel by default. |
| **Step** | Individual task: run command or use an action |
| **Action** | Reusable unit (e.g., actions/checkout@v4) |
| **Runner** | Machine executing jobs (GitHub-hosted or self-hosted) |

---

## COMPLETE CI/CD PIPELINE EXAMPLE

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger

env:
  REGISTRY: gcr.io/myproject
  IMAGE_NAME: myapp

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true

    - name: Run tests
      run: go test ./... -v -coverprofile=coverage.out

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run Trivy
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: fs
        severity: HIGH,CRITICAL
        exit-code: '1'

  build-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write  # For OIDC

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}

    steps:
    - uses: actions/checkout@v4

    - name: Authenticate to GCP (OIDC - no static keys!)
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
        service_account: github-actions@myproject.iam.gserviceaccount.com

    - name: Configure Docker for GCR
      run: gcloud auth configure-docker gcr.io

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,format=short
          type=semver,pattern={{version}}

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  update-gitops:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: myorg/gitops-repo
        token: ${{ secrets.GITOPS_TOKEN }}

    - name: Update image tag
      run: |
        IMAGE_TAG=$(echo '${{ needs.build-push.outputs.image-tag }}' | head -1)
        sed -i "s|image: .*myapp:.*|image: $IMAGE_TAG|" apps/myapp/deployment.yaml
        git config user.email 'actions@github.com'
        git config user.name 'GitHub Actions'
        git add .
        git commit -m 'ci: update myapp image to ${{ github.sha }}'
        git push
```

---

## ADVANCED FEATURES

### Reusable Workflows
```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      KUBECONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - name: Deploy
      run: helm upgrade --install myapp ./charts/myapp -n ${{ inputs.environment }}

# .github/workflows/production-deploy.yml
jobs:
  deploy-prod:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}
```

### Matrix Builds
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: ['1.20', '1.21', '1.22']
        os: [ubuntu-latest, macos-latest]
    steps:
    - uses: actions/setup-go@v5
      with:
        go-version: ${{ matrix.go-version }}
```

### Caching
```yaml
# Cache Go modules
- uses: actions/cache@v3
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-

# Docker layer caching (BuildKit)
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Environment Protection Rules
```yaml
# Production environment requires manual approval
jobs:
  deploy-prod:
    environment: production  # Environment with required reviewers in GitHub settings
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to production
      run: ./deploy.sh production
```

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: How is GitHub Actions different from Jenkins?

| Aspect | GitHub Actions | Jenkins |
|---|---|---|
| Hosting | GitHub SaaS | Self-hosted |
| Setup | Zero setup | Install + configure |
| Integration | Native GitHub | Plugin-based |
| Runners | Free minutes included | Your own infra |
| Marketplace | 20,000+ actions | 1800+ plugins |
| Secrets | GitHub Secrets | Jenkins credentials |
| Cost | Free for public repos, paid for private | Infra cost |

**When GitHub Actions**: GitHub-hosted projects, modern pipelines, simple to medium complexity.
**When Jenkins**: Complex enterprise pipelines, legacy system integration, on-premises requirement.

### Q2: How do you handle secrets in GitHub Actions?
```yaml
# Option 1: GitHub repository/environment secrets
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# Option 2: OIDC (best practice - no static credentials)
# Authenticates to AWS/GCP using temporary tokens
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: '...'
    service_account: 'github-actions@project.iam.gserviceaccount.com'

# Option 3: HashiCorp Vault
- uses: hashicorp/vault-action@v2
  with:
    url: https://vault.example.com
    role: github-actions
    secrets: secret/db password | DB_PASSWORD
```

**Best practice**: OIDC over static credentials. No secrets stored in GitHub at all.

### Q3: How do you optimize GitHub Actions pipeline speed?
1. **Caching**: Cache dependencies (Go modules, npm, pip, Docker layers)
2. **Parallelism**: Run jobs in parallel (test, security scan, lint)
3. **Matrix builds**: Build for multiple versions simultaneously
4. **Skip unnecessary jobs**: Use if conditions, path filters
5. **Smaller Docker base images**: Faster pulls
6. **Reusable workflows**: Share pipeline code across repos

### Q4: How do you prevent secret exposure in workflow logs?
- Secrets registered in GitHub Secrets are **automatically masked** in logs
- **Never** echo secrets directly: `echo $SECRET` → use `echo '***'`
- Use `add-mask` for dynamic secrets:
```yaml
- run: echo '::add-mask::${{ steps.get-token.outputs.token }}'
```
- Audit logs: GitHub provides audit trail of secret access

### Q5: Explain OIDC authentication in GitHub Actions
OIDC (OpenID Connect) allows GitHub Actions to authenticate to cloud providers without storing static credentials.

**Flow**:
1. Job requests OIDC token from GitHub
2. Token is short-lived, contains: repo name, branch, actor
3. Cloud provider (AWS/GCP) configured to trust GitHub OIDC
4. GitHub token exchanged for cloud credentials (AWS role, GCP SA)

**Benefits**: No long-lived credentials. Credentials scoped to branch/environment. Auto-expiry.

### Q6: How do you implement environment promotion?
```yaml
# Deploy to staging automatically, production with approval
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to staging
      run: helm upgrade --install myapp . -n staging

  deploy-production:
    needs: deploy-staging
    environment: production  # Requires manual approval in GitHub Settings
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to production
      run: helm upgrade --install myapp . -n production
```
