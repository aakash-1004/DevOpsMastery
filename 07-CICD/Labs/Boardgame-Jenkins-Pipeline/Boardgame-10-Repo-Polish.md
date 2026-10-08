---
title: PHASE 10 — Repo Polish
type: project
module: cicd
status: completed
tags:
  - jenkins
  - kubeadm
  - kubernetes
  - rbac
  - sonarqube
  - nexus
  - trivy
  - prometheus
  - grafana
  - blackbox-exporter
  - cicd
  - devops
  - notes
  - project
order: 16
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-09-Monitoring-Stack]]"
next: "[[Boardgame-11-Instance-IDs-Reference]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 10 — Repo Polish

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-09-Monitoring-Stack|PHASE 9 — Monitoring Stack]] · → [[Boardgame-11-Instance-IDs-Reference|Instance IDs — Full Reference]]
<!-- /nav -->

## Added RBAC manifests to the repo

Created `k8s/rbac/` directory containing serviceaccount.yaml, role.yaml, rolebinding.yaml, and secret-template.yaml (template only — does NOT contain the live token, since the real value is auto-populated server-side by Kubernetes).

## Updated README.md

Replaced the original placeholder with a comprehensive project README covering architecture diagram, pipeline stages, key engineering decisions, and the full tech stack.

## Added .dockerignore

```
.git
.gitignore
target
*.md
k8s
Jenkinsfile
```

Why, even though the Dockerfile uses selective COPY (not blanket COPY .): reduces build context transfer size (Docker sends everything not excluded to the daemon before reading any Dockerfile instructions), and acts as defense-in-depth against a future edit accidentally introducing a blanket COPY.

## Updated .gitignore

Added: `trivy-*-report.html` — prevents scan report artifacts from accidentally being committed if a scan is ever run locally in this directory.
