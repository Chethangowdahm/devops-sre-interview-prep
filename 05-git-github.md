# Git & GitHub — Deep Dive for SRE/DevOps (6 YOE)

---

## Git Core Concepts

```
Working Directory → Staging Area (Index) → Local Repo → Remote Repo
      │                    │                    │              │
   (edit files)        git add             git commit      git push
```

### Object Model
```
Blob   → file content
Tree   → directory (points to blobs + other trees)
Commit → snapshot (points to tree + parent commit + metadata)
Tag    → named pointer to a commit
Branch → movable pointer to a commit (HEAD is current branch pointer)
```

---

## Branching Strategies

### GitFlow (Complex, legacy)
```
main ──────────────────────────────────────► (production releases)
  └─ develop ────────────────────────────►  (integration branch)
         ├─ feature/auth ──────────────►    (feature work)
         ├─ release/1.2.0 ────────────►     (release prep)
         └─ hotfix/critical-bug ──────►     (emergency prod fix)
```

### Trunk-Based Development (Modern, recommended for SRE/DevOps)
```
main ────────────────────────────────────────► (always deployable)
  ├─ feature/add-login    (short-lived, <1 day ideally)
  ├─ feature/fix-timeout  (merged same day via PR)
  └─ feature/update-deps  (small, frequent merges)
```
> Trunk-based: smaller PRs, faster feedback, fewer merge conflicts, enables CI properly.

### GitHub Flow (Simple, cloud-native)
```
main → feature branch → PR → review → merge → auto-deploy
```

---

## Essential Git Commands

```bash
# ── Setup ───────────────────────────────────────────
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor vim
git config --global pull.rebase true        # rebase instead of merge on pull

# ── Daily workflow ───────────────────────────────────
git status
git diff                          # unstaged changes
git diff --staged                 # staged changes
git log --oneline --graph --all   # visual branch history
git stash push -m "WIP: fix auth"
git stash pop

# ── Branching ────────────────────────────────────────
git checkout -b feature/add-login    # create + switch
git branch -d feature/add-login      # delete local (safe)
git branch -D feature/add-login      # force delete
git push origin --delete feature/add-login  # delete remote

# ── Merging ──────────────────────────────────────────
git merge feature/auth               # creates merge commit
git merge --squash feature/auth      # squash all commits into one staged change
git rebase main                      # rebase current branch onto main (linear history)
git rebase -i HEAD~3                 # interactive rebase: squash/fixup/reorder

# ── Undoing ──────────────────────────────────────────
git restore file.txt                 # discard working dir changes
git restore --staged file.txt        # unstage
git reset HEAD~1                     # undo last commit, keep changes (soft-ish)
git reset --hard HEAD~1              # undo last commit, discard changes (DESTRUCTIVE)
git revert abc1234                   # create new commit that undoes abc1234 (safe for shared branches)
git cherry-pick abc1234              # apply a specific commit to current branch

# ── Remote ───────────────────────────────────────────
git remote -v
git fetch --all --prune              # fetch all remotes, remove deleted remote branches
git pull --rebase origin main
git push -u origin feature/add-login
git push --force-with-lease          # safer force push (fails if remote was updated)

# ── Tags (for releases) ──────────────────────────────
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0
git push origin --tags

# ── Troubleshooting ──────────────────────────────────
git bisect start                     # binary search for bug-introducing commit
git bisect bad                       # current commit is bad
git bisect good v1.1.0               # this commit was good
git blame file.py                    # who changed each line, when
git reflog                           # recover lost commits (30-day log)
git shortlog -sn                     # commit count by author
```

---

## GitHub Features for SRE/DevOps

### Pull Request Best Practices

```
Good PR:
  ✓ Small, focused change (<400 lines diff ideally)
  ✓ Descriptive title: "fix: handle nil pointer in auth middleware"
  ✓ Links to issue/ticket
  ✓ Description: what + why + how to test
  ✓ Screenshots/logs for UI/infrastructure changes

PR Description Template:
  ## What
  ## Why
  ## How to test
  ## Checklist
  - [ ] Tests added/updated
  - [ ] Docs updated
  - [ ] Runbook updated if behavior changes
```

### Branch Protection Rules
```
Settings → Branches → Branch protection rules → main:
  ✓ Require pull request reviews (minimum 1-2)
  ✓ Require status checks (CI must pass)
  ✓ Require branches to be up to date
  ✓ Require signed commits
  ✓ Include administrators
  ✗ Allow force pushes (disable!)
  ✗ Allow deletions (disable for main!)
```

### CODEOWNERS
```
# .github/CODEOWNERS
# Auto-request review from owners when their code is changed

*                   @org/all-devs          # default: all devs review
/infrastructure/    @org/sre-team          # SRE owns infra
/k8s/               @org/sre-team
*.go                @org/backend-team
*.ts                @org/frontend-team
/docs/              @org/tech-writers
```

---

## GitHub Actions — CI/CD

GitHub Actions is GitHub's built-in CI/CD. No Jenkins required.

```yaml
# .github/workflows/ci.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: gcr.io
  IMAGE: gcr.io/${{ secrets.GCP_PROJECT }}/myapp

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true

    - name: Run tests
      run: go test ./... -race -coverprofile=coverage.out

    - name: Upload coverage
      uses: codecov/codecov-action@v4

  build-push:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      id-token: write     # for Workload Identity Federation
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Authenticate to GCP
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
        service_account: ${{ secrets.SA_EMAIL }}

    - name: Configure Docker for GCR
      run: gcloud auth configure-docker gcr.io

    - name: Build and push
      run: |
        IMAGE_TAG="${{ env.IMAGE }}:${{ github.sha }}"
        docker build -t "$IMAGE_TAG" -t "${{ env.IMAGE }}:latest" .
        docker push "$IMAGE_TAG"
        docker push "${{ env.IMAGE }}:latest"
        echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_OUTPUT

    - name: Security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.IMAGE }}:${{ github.sha }}
        exit-code: '1'
        severity: 'HIGH,CRITICAL'

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build-push
    environment: production       # requires manual approval in GitHub
    steps:
    - name: Deploy to GKE
      run: |
        kubectl set image deployment/myapp \
          app=${{ env.IMAGE }}:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/myapp -n production --timeout=10m
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yaml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - name: Deploy
      run: kubectl set image deployment/myapp app=gcr.io/proj/myapp:${{ inputs.image_tag }}
```

---

## Git Hooks (Automation at commit level)

```bash
# .git/hooks/pre-commit (or use pre-commit framework)
#!/bin/sh
# Run before every commit

# Lint
golangci-lint run ./...
if [ $? -ne 0 ]; then
  echo "Lint failed. Fix errors before committing."
  exit 1
fi

# Check for secrets
git diff --cached | grep -iE "(password|secret|api_key|token)\s*=" && \
  echo "WARNING: Possible secret detected!" && exit 1

exit 0
```

Use `pre-commit` framework for managed hooks:
```yaml
# .pre-commit-config.yaml
repos:
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v4.5.0
  hooks:
  - id: detect-private-key
  - id: check-yaml
  - id: trailing-whitespace
- repo: https://github.com/dnephin/pre-commit-golang
  rev: v0.5.1
  hooks:
  - id: go-fmt
  - id: go-vet
```

---

## Git Commit Message Convention (Conventional Commits)

```
<type>(<scope>): <subject>

<body>

<footer>

Types:
  feat:     new feature
  fix:      bug fix
  perf:     performance improvement
  refactor: code change (no feature, no fix)
  test:     test changes
  docs:     documentation
  ci:       CI/CD changes
  chore:    build/tool changes

Examples:
  feat(auth): add OAuth2 Google login
  fix(api): handle nil pointer in user lookup
  perf(db): add index on user_id column
  ci: add trivy security scan to pipeline
```

---

## Interview Questions — Git/GitHub (6 YOE Level)

**Q: What's the difference between `git merge`, `git rebase`, and `git cherry-pick`?**
> **merge**: Combines two branch histories, creates a merge commit, preserves full history. **rebase**: Re-applies commits on top of another branch, creates linear history, rewrites commit hashes (don't rebase shared branches). **cherry-pick**: Apply a specific commit from one branch to another — useful for backporting hotfixes.

**Q: How do you handle merge conflicts in a team setting?**
> Use short-lived branches, merge frequently. When conflict occurs: `git fetch`, `git rebase origin/main` (or merge), resolve conflicts in each file, `git add`, `git rebase --continue`. Review the conflict carefully — don't just accept either side blindly. Use `git mergetool` or IDE conflict resolution. Prevention: small PRs, frequent integration.

**Q: What is `git reflog` and when would you use it?**
> `reflog` tracks every movement of HEAD for ~30 days, even after resets, rebases, or branch deletions. Use it to recover "lost" commits after a bad `git reset --hard` or accidental branch deletion: `git reflog` → find the commit SHA → `git checkout -b recovery SHA`.

**Q: How do you implement a hotfix workflow?**
```bash
# 1. Branch from the production tag/commit
git checkout -b hotfix/critical-null-ptr v1.2.0
# 2. Fix, test, commit
git commit -m "fix: handle null ptr in payment processor"
# 3. PR to main → merge
# 4. Tag the fix
git tag -a v1.2.1 -m "Hotfix: null ptr in payment"
# 5. Backport to release branches if needed
git checkout release/1.2 && git cherry-pick <fix-sha>
```

**Q: What's the difference between `git reset` and `git revert`?**
> `reset` moves the branch pointer backward, rewriting history (dangerous for shared branches). `revert` creates a new commit that undoes the changes, preserving history (safe for shared branches). For production hotfixes on main, always use `revert` to maintain audit trail.
