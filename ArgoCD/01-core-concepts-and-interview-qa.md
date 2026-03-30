# ArgoCD - Core Concepts, Interview Q&A & Scenarios
> Senior DevOps/SRE Level | GitOps Expert

---

## CORE CONCEPTS

### What is ArgoCD?
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It monitors Git repositories and automatically syncs the cluster state to match what's defined in Git.

**GitOps Principle**: Git is the single source of truth. All changes go through Git. ArgoCD ensures the cluster always matches Git.

### ArgoCD Architecture
- **API Server**: gRPC/REST server. Handles auth, app management, RBAC.
- **Repository Server**: Clones Git repos, generates K8s manifests (Helm, Kustomize, plain YAML, Jsonnet).
- **Application Controller**: Compares live state vs desired state, syncs, manages health.
- **Dex**: SSO/OIDC integration.
- **Redis**: Caching.

### Key Objects

**Application**: Core resource. Defines what to deploy and where.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: HEAD
    path: apps/my-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Revert manual changes to cluster
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        maxDuration: 3m
        factor: 2
```

**ApplicationSet**: Generates multiple Applications from templates (for multi-cluster, multi-env).
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-envs
spec:
  generators:
  - list:
      elements:
      - cluster: staging
        url: https://staging-cluster
      - cluster: production
        url: https://prod-cluster
  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        path: apps/my-app/overlays/{{cluster}}
```

**AppProject**: Groups applications, defines allowed sources/destinations, RBAC.

### Sync Strategies
- **Manual sync**: Review plan, click sync or use CLI
- **Auto sync**: ArgoCD syncs automatically when Git changes
- **Auto-prune**: Remove resources deleted from Git
- **Self-heal**: Revert manual changes to cluster (drift correction)

### App Health Status
- **Healthy**: All resources running as expected
- **Progressing**: Deployment rolling out
- **Degraded**: Resource not in desired state
- **Suspended**: Rollout suspended
- **Missing**: Expected resource not found
- **Unknown**: Can't determine health

### Sync Status
- **Synced**: Cluster matches Git
- **OutOfSync**: Cluster differs from Git (drift)
- **Unknown**: Can't compare

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: What is GitOps and how does ArgoCD implement it?
GitOps is a practice where Git is the single source of truth for infrastructure and application config. All changes are made via Git commits/PRs.

**ArgoCD implements GitOps by**:
1. Watching Git repos for changes
2. Comparing desired state (Git) with live state (cluster)
3. Automatically syncing when out-of-sync (auto-sync)
4. Self-healing (reverting manual cluster changes back to Git state)
5. Providing audit trail — every deployment is a Git commit

**Benefits at Vendasta**: Reduced manual deployment errors by 70%. Full audit trail. Easy rollback (git revert). Declarative, version-controlled delivery.

### Q2: How does ArgoCD handle image updates? (Typical workflow with CI/CD)
ArgoCD doesn't build images — it deploys them. The CI/CD pipeline updates image tags in Git, and ArgoCD syncs.

**Full GitOps workflow**:
```
1. Developer pushes code to application repo
2. CI pipeline (Jenkins/GitHub Actions) builds and pushes image to registry
   docker push gcr.io/myproject/myapp:git-abc1234
3. CI pipeline updates image tag in GitOps repo (separate repo):
   - Updates Helm values.yaml or Kustomize image override
   - Creates PR or commits directly
4. ArgoCD detects change in GitOps repo
5. ArgoCD syncs cluster with new image tag
6. Health checks pass → deployment complete
```

**Alternative**: ArgoCD Image Updater (watches registry, auto-updates GitOps repo).

### Q3: How do you handle secrets with ArgoCD/GitOps?
**NEVER put secrets in Git** (even encrypted base64 is risky).

**Approaches**:
1. **Sealed Secrets** (Bitnami): Encrypt secrets with cluster's public key. Safe to store in Git. Only cluster can decrypt.
2. **External Secrets Operator**: Pulls from AWS Secrets Manager/GCP Secret Manager at runtime. Secret reference in Git, not value.
3. **Vault + ArgoCD Vault Plugin**: Secrets fetched from Vault during sync.
4. **SOPS**: Encrypt files with GPG/KMS. Decrypt at apply time.

**Production choice**: External Secrets Operator + GCP Secret Manager. Secrets rotate independently. No secrets in Git.

### Q4: How do you do a rollback in ArgoCD?
```bash
# Option 1: Roll back to previous sync (via CLI)
argocd app history myapp
argocd app rollback myapp <revision-id>

# Option 2: Git revert (preferred GitOps approach)
git revert abc1234  # Reverts the bad commit
git push origin main
# ArgoCD auto-syncs with reverted state

# Option 3: ArgoCD UI → History and Rollback tab
```

**Why git revert is preferred**: It creates an audit trail. The rollback itself is recorded in Git.

### Q5: What is the difference between syncing and refreshing in ArgoCD?
- **Refresh**: ArgoCD re-fetches the Git repo and compares with live state. Updates OutOfSync status. No changes to cluster.
- **Sync**: Actually applies changes to cluster. Deploys what's in Git.

### Q6: How do you handle multiple environments/clusters with ArgoCD?
**Options**:
1. **ApplicationSet with cluster generator**: Deploy same app to multiple clusters
2. **App of Apps pattern**: One parent Application manages child Applications
3. **Separate directories**: apps/myapp/overlays/staging, apps/myapp/overlays/production
4. **Separate repos**: GitOps repo per environment (more isolation)

**App of Apps pattern (used at Vendasta)**:
```
gitops-repo/
├── apps/                    # Parent app manages these
│   ├── api-service/
│   ├── worker-service/
│   └── frontend/
└── argocd-apps/             # Parent Application manifests
    ├── api-service-app.yaml
    └── worker-service-app.yaml
```

### Q7: How do you set up RBAC in ArgoCD?
```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data:
  policy.csv: |
    # Developers: read-only in prod, sync in staging
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, staging/*, allow

    # SRE team: full access
    p, role:sre, applications, *, */*, allow
    p, role:sre, clusters, *, *, allow

    # Group assignments
    g, my-org:developers, role:developer
    g, my-org:sre, role:sre
  policy.default: role:readonly
```

### Q8: Sync Hooks - pre/post sync operations
```yaml
# Run database migration before sync
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest
        command: ["./migrate", "up"]
      restartPolicy: Never
```

**Hook types**: PreSync, Sync, PostSync, SyncFail, PostDelete

---

## REAL-WORLD SCENARIOS

### Scenario: ArgoCD Out of Sync — Manual Changes Made
Someone ran `kubectl set image deployment/myapp app=myapp:hotfix` directly on cluster.
ArgoCD shows OutOfSync.

**Options**:
1. **Accept drift temporarily**: ArgoCD won't revert (selfHeal=false)
2. **Force sync**: ArgoCD reverts to Git state (removes hotfix)
3. **Update Git**: Update image tag in Git repo, then sync
4. **GitOps discipline**: Create emergency PR with hotfix, merge, ArgoCD syncs

**Production practice**: If self-heal is enabled, ArgoCD reverts within sync interval (3min default). For genuine hotfixes, go through Git.

### Scenario: Deployment Stuck in Progressing
```bash
# Check why
argocd app get myapp --show-operation
kubectl describe application myapp -n argocd
kubectl get pods -n production

# If new pods stuck (ImagePullBackOff, CrashLoopBackOff):
kubectl describe pod <new-pod> -n production

# Rollback if needed
argocd app rollback myapp
# Or git revert
```
