---
title: "Revision — GitOps & ArgoCD"
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 9
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-08-CICD-GitHub-Actions-and-Jenkins]]"
next: "[[Revision-10-Monitoring-Prometheus-and-Grafana]]"
related:
  - "[[7.3-ArgoCD-and-GitOps]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — GitOps & ArgoCD

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-08-CICD-GitHub-Actions-and-Jenkins|Revision — CI/CD: GitHub Actions & Jenkins]] · → [[Revision-10-Monitoring-Prometheus-and-Grafana|Revision — Monitoring: Prometheus & Grafana]]
<!-- /nav -->

## Concept

GitOps is an operating model where Git is the **single source of truth** for both application code and infrastructure state. All changes flow through Git (PR, review, merge). The actual deployment is automated by an agent running in the cluster that continuously reconciles cluster state to match what's in Git.

**The GitOps principle:**

- Desired state = what's in Git
- Actual state = what's in the cluster
- ArgoCD continuously detects drift (actual ≠ desired) and auto-syncs

**Why this matters:**

- Full audit trail: every change is a Git commit (who, what, when, why)
- Easy rollback: `git revert` + sync
- Self-healing: if someone does `kubectl edit` manually, ArgoCD reverts it
- No kubectl access needed for deployments — Git is the deploy interface

---

## Core ArgoCD Commands

```bash
# Installation on k3s
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts   # Needed for large CRDs

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ArgoCD CLI
argocd login localhost:8080
argocd app list
argocd app sync myapp
argocd app status myapp
argocd app history myapp
argocd app rollback myapp 5
```

**ArgoCD Application manifest:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fullstack-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/aakash-1004/fullstack-chart
    targetRevision: main
    path: fullstack-chart                # Path to Helm chart in repo
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Revert manual changes in cluster
    syncOptions:
      - CreateNamespace=true
```

---

## Senior Interview Q&A

**Q: What is GitOps and how does it differ from traditional CI/CD?**

> "In traditional CI/CD, the pipeline has write access to the cluster — it runs `kubectl apply` or `helm upgrade` as part of the deploy stage. The pipeline is the deployment mechanism.
>
> In GitOps, the pipeline never touches the cluster directly. Instead, it updates the Git repo (a 'config repo' or updates the image tag in a Helm values file). An agent running inside the cluster (ArgoCD, Flux) watches Git and reconciles the cluster state to match.
>
> The inversion is significant: instead of push-based deployment (pipeline pushes to cluster), you have pull-based deployment (cluster agent pulls from Git).
>
> Operational benefits: you can revoke all external CI/CD cluster access — nothing from outside needs to deploy. The cluster manages itself. Full audit trail in Git. Disaster recovery becomes `git clone + argocd sync` — you can recreate a cluster from Git state.
>
> The trade-off: it adds complexity. You need to manage the separation between 'app code repo' and 'config repo', and the CI pipeline needs to update image tags in the config repo as part of its flow."

---

**Q: ArgoCD shows an application is OutOfSync. What does that mean and how do you handle it?**

> "OutOfSync means what ArgoCD sees in the cluster (actual state) doesn't match what's in Git (desired state).
>
> This can happen three ways:
>
> 1. Someone changed something directly in the cluster via `kubectl` or Helm — a manual change that bypassed Git.
> 2. You pushed a change to the Git config repo and ArgoCD hasn't synced yet (or auto-sync is off).
> 3. A resource was created outside ArgoCD's management and conflicts with a managed resource.
>
> If auto-sync and selfHeal are enabled, ArgoCD will reconcile automatically within seconds.
>
> If manual: click Sync in the UI or `argocd app sync myapp`. It applies the Git state to the cluster.
>
> Investigation: `argocd app diff myapp` shows exactly what's different. This is your first step — understand the diff before you sync, to avoid overwriting something important.
>
> In production: always have `selfHeal: true`. Any manual kubectl edit should be treated as a drift event, logged, and reverted. If you need to make an emergency change, make it in Git — even if it takes 5 minutes longer, you maintain auditability."

<!-- related -->
---
**Related:** [[7.3-ArgoCD-and-GitOps|CI/CD: ArgoCD & GitOps]]
<!-- /related -->
