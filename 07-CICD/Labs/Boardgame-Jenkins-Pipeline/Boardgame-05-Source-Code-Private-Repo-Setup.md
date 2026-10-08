---
title: "PHASE 5 — Source Code: Private Repo Setup"
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
order: 11
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-04-SonarQube-and-Postgres-Server-Setup]]"
next: "[[Boardgame-06-RBAC-Jenkins-to-Kubernetes-Access]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 5 — Source Code: Private Repo Setup

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-04-SonarQube-and-Postgres-Server-Setup|PHASE 4 — SonarQube + Postgres Server Setup]] · → [[Boardgame-06-RBAC-Jenkins-to-Kubernetes-Access|PHASE 6 — RBAC: Jenkins → Kubernetes Deployment Access]]
<!-- /nav -->

## Cloned reference app, stripped history, pushed to own private repo

```bash
cd ~/aakash/devops/devops-project
git clone https://github.com/jaiswaladi246/Boardgame.git boardgame-jenkins-cicd-pipeline
cd boardgame-jenkins-cicd-pipeline
rm -rf .git
git init
git add .
git commit -m "Initial commit: Boardgame app source (reference Jenkinsfile included, will be rewritten)"
git branch -M main
git remote add origin https://github.com/aakash-1004/boardgame-jenkins-cicd-pipeline.git
git push -u origin main
```

Why rm -rf .git + git init: removes the original repo's entire commit history and link back to the creator's repo — makes this a genuinely fresh project, not a visible fork.

## Modified pom.xml

Changed Maven repository URLs in `<distributionManagement>` to point at our own Nexus server's maven-releases and maven-snapshots repositories.
