# CI/CD Pipeline: GitOps Validation & Deployment

## Overview

Pure GitOps CI/CD pipeline using Forgejo Actions (self-hosted runner).

**Principle:** Validate in CI, deploy via ArgoCD (no manual steps).

```
git push
  ↓
  [CI: Validate]
    ├─ yamllint (YAML syntax)
    ├─ kubeval (K8s manifests)
    ├─ kustomize build (all layers)
    ├─ argocd validation (app definitions)
    └─ security scan (secrets, best practices)
  ↓
  [If push to main]
    └─ ArgoCD auto-syncs (if enabled)
```

## Workflows

### 1. validate-k8s.yaml (Mandatory)

**Trigger:** Any push/PR with k8s/ changes

**What it does:**
1. Lints all YAML files (`yamllint`)
2. Validates K8s manifests (`kubeval`)
3. Builds all kustomization layers
4. Validates ArgoCD applications
5. Reports results

**Duration:** ~2-3 minutes

**Status:**
- ✅ PASS: All layers build, manifests valid → OK to merge
- ❌ FAIL: Syntax error, invalid resource, build failed → Fix & push again

**Example output:**
```
=== Building k8s/infrastructure/ ===
✓ Infrastructure built successfully
Resources: 47

=== Building k8s/bootstrap/ ===
✓ Bootstrap built successfully
Resources: 23
```

**When to check:**
- After every commit
- Before merging PRs
- On every branch

### 2. argocd-sync.yaml (Recommended)

**Trigger:** Push to main only (k8s/ changed)

**What it does:**
1. Authenticates with ArgoCD
2. Syncs `homelab-root` application
3. Waits for sync to complete (5 min timeout)
4. Verifies all applications healthy

**Duration:** 1-5 minutes (depends on resources)

**Status:**
- ✅ SYNCED: All resources deployed to cluster
- ❌ FAILED: Sync error, pod crashes, etc. → Check ArgoCD UI for details

**When it runs:**
- Automatically after merge to main
- Only on k8s/ changes (not on docs)

**Manual trigger (if needed):**
```bash
# SSH to runner or use Forgejo UI
# Re-run failed workflow
# Or manually sync: argocd app sync homelab-root
```

**Requires secrets:**
- `ARGOCD_SERVER`: ArgoCD server URL (https://argocd.riotpiao.com)
- `ARGOCD_AUTH_TOKEN`: ArgoCD API token (generate via ArgoCD UI)

### 3. security-scan.yaml (Optional)

**Trigger:** Any push/PR with k8s/ changes

**What it does:**
1. Scans Dockerfiles for vulnerabilities (`trivy`)
2. Scans Helm charts for security issues
3. Audits K8s manifests (`polaris`)
4. Checks for hardcoded secrets
5. Verifies security best practices

**Duration:** ~3-5 minutes

**Status:**
- ✅ PASS: No critical issues
- ⚠️ WARNING: Best practice recommendations (non-blocking)
- ❌ FAIL: Hardcoded secrets found (must fix)

**Common issues:**
- Missing resource limits (warning)
- Privileged containers (warning)
- Hardcoded passwords (ERROR)

---

## File Structure

```
.forgejo/
├── workflows/                      # CI/CD workflows
│   ├── validate-k8s.yaml          # Validate manifests (required)
│   ├── argocd-sync.yaml           # Sync to cluster (auto on main)
│   └── security-scan.yaml         # Security checks (optional)
└── CI-CD.md                        # This file
```

---

## Setup Instructions

### 1. Install Forgejo Runner

```bash
# On runner machine (inside cluster or external)
forgejo-runner register \
  --instance https://forgejo.riotpiao.com \
  --token <registration-token> \
  --name homelab-runner \
  --labels docker

forgejo-runner daemon
```

### 2. Add ArgoCD Secrets to Forgejo

```bash
# Go to: Forgejo → Settings → Secrets

# Add:
ARGOCD_SERVER = https://argocd.riotpiao.com
ARGOCD_AUTH_TOKEN = <token>  # Generate: argocd account generate-token
```

### 3. Generate ArgoCD Token

```bash
# Inside cluster
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Go to: https://localhost:8080/user-info/api-tokens
# Create new token (CI/CD)
# Copy token to Forgejo secrets
```

---

## Workflow Execution

### When developer pushes to feature branch:

```
git push origin feature/new-service

↓
Forgejo Actions triggered
↓
validate-k8s.yaml runs:
  ✓ Lints YAML
  ✓ Validates manifests
  ✓ Builds kustomizations
  ✓ All pass → GitHub comment: "Ready to merge"
↓
Developer opens PR
↓
Reviewer checks:
  - Code changes (YAML)
  - Workflow results
  - ArgoCD impact (diff)
↓
PR merged to main
```

### When merged to main:

```
git merge feature/new-service → main

↓
Forgejo Actions triggered
↓
validate-k8s.yaml runs:
  ✓ Same validation as above
↓
argocd-sync.yaml runs (if enabled):
  ✓ Syncs homelab-root
  ✓ Waits for sync
  ✓ Verifies health
  ✓ Resources deployed to cluster
↓
Cluster state = git state
(No manual kubectl apply needed!)
```

---

## Debugging CI/CD Failures

### Issue: "Kustomize build failed"

```bash
# Run locally
cd k8s/
kustomize build bootstrap/  # See actual error

# Fix YAML/kustomization.yaml
# git push again
```

### Issue: "Kubeval validation failed"

```bash
# Check K8s manifest syntax
kubeval k8s/platform/minio/config.yaml

# Common issues:
# - Typos in apiVersion, kind, metadata
# - Missing required fields
# - Invalid references (namespace, service name)
```

### Issue: "ArgoCD sync failed"

```bash
# Check ArgoCD UI
# https://argocd.riotpiao.com → homelab-root

# Or CLI
argocd app get homelab-root
argocd app logs homelab-root --follow

# Common issues:
# - Missing namespace (fixed by infrastructure layer)
# - Invalid Helm chart version
# - Secret not found
# - Network policy blocking traffic
```

### Issue: "Security scan found hardcoded secret"

```bash
# Fix: Remove secret from YAML
# Add to SOPS encryption instead

# Or use ArgoCD Sealed Secrets
# (if SOPS not available)
```

---

## Viewing Results

### Forgejo Actions UI

```
Repository → Actions
  ├─ validate-k8s
  │   ├─ ✅ Success (merge safe)
  │   ├─ ❌ Failed (fix required)
  │   └─ Logs (click "Steps" → "Summary")
  ├─ argocd-sync
  │   ├─ ✅ Synced (deployed)
  │   └─ ❌ Failed (check ArgoCD UI)
  └─ security-scan
      ├─ ✅ Pass (no critical issues)
      └─ ⚠️  Warning (review, non-blocking)
```

### ArgoCD UI

```
https://argocd.riotpiao.com
  ├─ homelab-root
  │   ├─ Status: Synced ✓
  │   ├─ Health: Healthy ✓
  │   └─ Details (click to see resources)
  ├─ layer-1-bootstrap
  ├─ layer-2-platform
  ├─ layer-3-security
  ├─ layer-4-applications
  └─ layer-5-data
```

---

## Common Tasks

### Add new service to cluster

```bash
# 1. Create directory and kustomization.yaml
mkdir -p k8s/applications/my-service
cat > k8s/applications/my-service/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: my-namespace
helmCharts:
- name: my-chart
  repo: https://charts.example.com
  version: 1.0.0
  releaseName: my-service
  valuesFile: values.yaml
EOF

# 2. Add values.yaml
cp /template/values.yaml k8s/applications/my-service/

# 3. Commit and push
git add k8s/applications/my-service/
git commit -m "feat(apps): add my-service"
git push

# 4. CI validates
# 5. Merge to main
# 6. ArgoCD syncs automatically
# ✓ Service deployed to cluster
```

### Rollback a deployment

```bash
# 1. Find broken commit
git log --oneline k8s/  # Identify bad commit

# 2. Revert
git revert <commit-hash>
git push

# 3. CI validates (should pass)
# 4. Merge to main
# 5. ArgoCD syncs back to previous version
# ✓ Cluster state reverted
```

### Emergency: Disable ArgoCD auto-sync

```bash
# If production broken and need time to debug:
argocd app set homelab-root --sync-policy none

# Fix issue in git
# Test locally: kustomize build k8s/

# Re-enable
argocd app set homelab-root --sync-policy automated
argocd app sync homelab-root
```

---

## Monitoring & Alerts

### Check workflow status in Forgejo

```bash
# Dashboard shows:
✅ All green → Safe to merge
❌ Red → Fix required before merge
⏳ Yellow → Still running (wait)
```

### Check ArgoCD status

```bash
argocd app list
# Shows: Synced, OutOfSync, Unknown status

argocd app get homelab-root
# Shows: health, sync status, resources

argocd app logs homelab-root --follow
# Real-time logs during sync
```

### Alerts (optional, future)

```yaml
# Could add Forgejo webhooks → Slack/email
# When CI/CD fails → Alert ops team
# When ArgoCD goes OutOfSync → Alert ops team
```

---

## Troubleshooting

### Workflow doesn't trigger

**Check:**
- Is Forgejo runner running? `forgejo-runner daemon`
- Did you push to correct branch? (validate runs on all, argocd-sync only on main)
- Did path match filter? (must change k8s/ or .forgejo/workflows/)

### Workflow hangs/times out

**Check:**
- kustomize build → Check for dependency cycles
- argocd sync → Check cluster resources (storage full? network down?)
- security scan → Large image scan → Takes time

**Fix:**
- Increase timeout in workflow
- Optimize kustomization (remove unused resources)
- Add resource limits to pods

### ArgoCD token invalid

**Fix:**
```bash
# Regenerate token
argocd account generate-token

# Update Forgejo secret
# Settings → Secrets → ARGOCD_AUTH_TOKEN = <new-token>
```

---

## Best Practices

✅ **DO:**
- Commit all K8s changes to git (no manual kubectl apply)
- Run validate-k8s locally before push
- Write descriptive commit messages (why this change?)
- Review workflow logs before merging
- Monitor ArgoCD sync after merge

❌ **DON'T:**
- Push directly to main (always use PR)
- Skip workflow validation (it catches errors early)
- Ignore security scan warnings
- Manually `kubectl apply` (breaks GitOps)
- Edit resources in cluster (they revert via ArgoCD)

---

## Next Steps

1. **Setup Forgejo runner** (if not already running)
2. **Add ArgoCD secrets** to Forgejo
3. **Test workflows** on feature branch
4. **Merge to main** → Watch ArgoCD sync
5. **Celebrate:** Full GitOps pipeline working! 🎉
