# Setup Guide for Instructors

This guide explains how to integrate this GitOps repository with your Kubernetes cluster.

## Architecture Overview

```
┌──────────────────────────┐
│   GitHub Repository      │
│  ziyotek-edu/            │
│  container-gitops        │
│                          │
│  students/week-01.yaml   │◄─── Students submit PRs
└────────────┬─────────────┘
             │
             │ On merge to main
             ▼
┌──────────────────────────┐
│   GitHub Actions         │
│  - Validate YAML         │
│  - Generate manifests    │
│  - Commit to repo        │
└────────────┬─────────────┘
             │
             │ GitOps sync
             ▼
┌──────────────────────────┐
│   Kubernetes Cluster     │
│  namespace: container-   │
│    course-week01         │
│                          │
│  Flux/ArgoCD watches     │
│  manifests/generated/    │
└──────────────────────────┘
```

## Prerequisites

- Kubernetes cluster with GitOps tool (ArgoCD or Flux CD)
- Cilium Gateway API configured (or other ingress)
- Student router already deployed (from gitops-homelab)

## Option 1: ArgoCD (Recommended)

### Quick Deploy

This repository includes a pre-configured ArgoCD Application manifest:

\`\`\`bash
kubectl apply -f argocd/application.yaml
\`\`\`

This will:
- Create an ArgoCD Application named `container-course-students`
- Watch `manifests/generated/` in this repo
- Automatically sync changes (automated sync enabled)
- Prune removed resources
- Self-heal if manifests drift

### Verify Deployment

\`\`\`bash
# Check Application status
kubectl get application container-course-students -n argocd

# Force refresh (to pick up latest commits immediately)
kubectl -n argocd annotate application container-course-students \
  argocd.argoproj.io/refresh=normal --overwrite

# View detailed status
kubectl describe application container-course-students -n argocd
\`\`\`

### Integration with gitops-homelab

If you manage your cluster with a gitops-homelab repo, you can use the "App of Apps" pattern:

\`\`\`yaml
# In gitops-homelab/apps/base/container-course/argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: container-course-students
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ziyotek-edu/container-gitops
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
\`\`\`

This references the `argocd/` directory which contains the Application manifest.

## Option 2: Flux CD

### 1. Create GitRepository Source

In your `gitops-homelab` repo, add:

\`\`\`yaml
# gitops-homelab/apps/base/container-course/gitops-source.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: container-gitops
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/ziyotek-edu/container-gitops
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: container-course-students
  namespace: flux-system
spec:
  interval: 5m
  path: ./manifests/generated
  prune: true
  sourceRef:
    kind: GitRepository
    name: container-gitops
  targetNamespace: container-course-week01
\`\`\`

### 2. Apply to Cluster

\`\`\`bash
kubectl apply -f gitops-homelab/apps/base/container-course/gitops-source.yaml
\`\`\`

Flux will now:
- Watch the `container-gitops` repo
- Sync `manifests/generated/` to the cluster
- Auto-deploy student containers when PRs are merged

## Manual Testing

To test manifest generation locally:

\`\`\`bash
# Clone the repo
git clone https://github.com/ziyotek-edu/container-gitops
cd container-gitops

# Generate manifests
./scripts/generate-manifests.sh students/week-01.yaml

# Review generated files
ls -la manifests/generated/

# Apply to cluster (manual)
kubectl apply -k manifests/generated/
\`\`\`

## Adding Auto-Generation to CI

Add this job to `.github/workflows/validate-pr.yaml`:

\`\`\`yaml
generate-and-commit:
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main'
  needs: validate

  steps:
    - uses: actions/checkout@v4

    - name: Generate manifests
      run: |
        chmod +x scripts/generate-manifests.sh
        ./scripts/generate-manifests.sh students/week-01.yaml

    - name: Commit generated manifests
      run: |
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add manifests/generated/
        git diff --staged --quiet || git commit -m "Auto-generate manifests from week-01.yaml"
        git push
\`\`\`

This will automatically regenerate manifests when PRs are merged to main.

## Monitoring Deployments

Check deployment status:

\`\`\`bash
# List all student deployments
kubectl get deployments -n container-course-week01 -l app=student-app

# Check specific student
kubectl get pods -n container-course-week01 -l student=johndoe

# View logs
kubectl logs -n container-course-week01 -l student=johndoe

# Check gallery data
kubectl get configmap gallery-nginx-config -n container-course-week01 -o yaml
\`\`\`

## Troubleshooting

### Student container not deploying

1. Check if image is public: `docker manifest inspect ghcr.io/username/repo:tag`
2. Check deployment status: `kubectl describe deployment student-USERNAME -n container-course-week01`
3. Check pod logs: `kubectl logs -l student=USERNAME -n container-course-week01`

### Gallery not showing students

1. Check ConfigMap: `kubectl get cm gallery-nginx-config -n container-course-week01 -o yaml`
2. Restart gallery: `kubectl rollout restart deployment gallery -n container-course-week01`

### GitOps not syncing

**Flux:**
\`\`\`bash
flux reconcile source git container-gitops
flux reconcile kustomization container-course-students
\`\`\`

**ArgoCD:**
\`\`\`bash
argocd app sync container-course-students
\`\`\`
