# Deployment Options: Self-Hosted Runner vs Polling

This document compares two approaches for deploying student containers.

## Option 1: Self-Hosted Runner + Instant Sync (Recommended)

**How it works:**
```
Student PR merged → GHA self-hosted runner → Generate manifests →
Commit to repo → Trigger ArgoCD sync → Instant deployment
```

**Pros:**
- ✅ **Instant deployment** (no 3-minute wait)
- ✅ Runs in your homelab (has cluster access)
- ✅ No external secrets needed
- ✅ Direct kubectl access to ArgoCD
- ✅ Deployment feedback in GHA logs
- ✅ Students see deployment status immediately

**Cons:**
- ⚠️ Requires self-hosted runner (you already have ARC)
- ⚠️ Runner needs kubectl access to ArgoCD namespace

**Implementation:**
```yaml
# .github/workflows/deploy-manifests.yaml
jobs:
  generate-and-deploy:
    runs-on: self-hosted  # Your homelab-runners
    steps:
      - Generate manifests
      - Commit to repo
      - Trigger ArgoCD sync via kubectl
```

**Status:** ✅ Implemented in `.github/workflows/deploy-manifests.yaml`

---

## Option 2: ArgoCD Polling (Current Default)

**How it works:**
```
Student PR merged → GHA generates manifests → Commit to repo →
ArgoCD polls every 3 min → Eventually syncs → Delayed deployment
```

**Pros:**
- ✅ Simple setup
- ✅ No runner cluster access needed
- ✅ ArgoCD handles everything automatically

**Cons:**
- ⚠️ 3-minute delay (default polling interval)
- ⚠️ No immediate feedback
- ⚠️ Students wait longer to see their deployment

**Current Configuration:**
```yaml
# argocd/application.yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    # No refresh annotation - relies on polling
```

---

## Option 3: Hybrid (GitHub Webhook → ArgoCD)

**How it works:**
```
Student PR merged → GHA generates manifests → GitHub webhook →
ArgoCD webhook receiver → Instant sync
```

**Pros:**
- ✅ Instant deployment
- ✅ No self-hosted runner needed
- ✅ ArgoCD native feature

**Cons:**
- ⚠️ Requires exposing ArgoCD webhook endpoint
- ⚠️ Need to configure webhook secret
- ⚠️ More complex firewall/ingress setup

**Not implemented** (requires network exposure)

---

## Recommendation

**Use Option 1 (Self-Hosted Runner)** because:

1. You already have ARC runners (`homelab-runners`)
2. Your runner has cluster access
3. Instant feedback for students
4. No additional infrastructure needed
5. Simple kubectl commands

---

## Testing the Self-Hosted Runner Workflow

### 1. Verify runner has access:

```bash
# On your runner pod, test:
kubectl get application container-course-students -n argocd
```

### 2. Test the workflow:

```bash
# Make a change to students/week-01.yaml
cd ~/git/container-gitops
echo "  # Test comment" >> students/week-01.yaml
git add students/week-01.yaml
git commit -m "Test: trigger workflow"
git push origin main
```

### 3. Watch the workflow:

- GitHub Actions: https://github.com/ziyotek-edu/container-gitops/actions
- ArgoCD Application status:
  ```bash
  kubectl get application container-course-students -n argocd -w
  ```

### 4. Verify instant deployment:

```bash
# Should update within seconds, not minutes
kubectl get pods -n container-course-week01 -l app=student-app -w
```

---

## Troubleshooting

### Runner doesn't have kubectl access

If the self-hosted runner can't access kubectl:

```bash
# Check runner pod
kubectl get pods -n arc-systems

# Verify ServiceAccount has ArgoCD access
kubectl get rolebinding -n argocd | grep runner
```

**Solution:** Create RBAC for runner:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-refresh-role
  namespace: argocd
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["applications"]
    verbs: ["get", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: runner-argocd-refresh
  namespace: argocd
subjects:
  - kind: ServiceAccount
    name: <runner-service-account>
    namespace: arc-systems
roleRef:
  kind: Role
  name: argocd-refresh-role
  apiGroup: rbac.authorization.k8s.io
```

### Workflow fails to push commits

The workflow uses `GITHUB_TOKEN` which has automatic push access. If it fails:

```bash
# Check token permissions in repo Settings > Actions > General
# Ensure "Read and write permissions" is enabled
```

### ArgoCD not syncing after annotation

```bash
# Manually trigger sync
kubectl -n argocd annotate application container-course-students \
  argocd.argoproj.io/refresh=hard --overwrite

# Check Application events
kubectl describe application container-course-students -n argocd
```

---

## Switching Between Options

### Enable polling-only (disable self-hosted sync):

1. Disable the `deploy-manifests.yaml` workflow
2. ArgoCD will fall back to 3-minute polling

### Re-enable instant sync:

1. Re-enable the workflow
2. Push a change to trigger it
