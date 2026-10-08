---
title: PHASE 8 — Pipeline Runs and Real Debugging
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
order: 14
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-07-Jenkinsfile-Writing-Debugging-Correction]]"
next: "[[Boardgame-09-Monitoring-Stack]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 8 — Pipeline Runs and Real Debugging

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-07-Jenkinsfile-Writing-Debugging-Correction|PHASE 7 — Jenkinsfile: Writing, Debugging, and Correction]] · → [[Boardgame-09-Monitoring-Stack|PHASE 9 — Monitoring Stack]]
<!-- /nav -->

## First real run — 8 stages passed, failed at Docker build

Successful: Git Checkout, Compile, Test (6 tests, 0 failures), File System Scan, SonarQube Analysis, Quality Gate, Build, Publish to Nexus (real 48MB jar uploaded to Nexus successfully). Failed at Docker build: `docker login` error — "client version 1.29 is too old. Minimum supported API version is 1.40."

Root cause: `withDockerRegistry`'s `toolName: 'docker'` auto-downloaded an ancient Docker CLI from a legacy distribution URL, completely separate from the modern docker-ce already installed. Fixed via Jenkins Global Tool Configuration — pointed the `docker` tool at `/usr` (Installation root), unchecked "Install automatically."

## Second run — Quality Gate failure (genuine enforcement)

SonarQube gate failed on 79.62% code duplication (Trivy report files analyzed as source). Pipeline correctly aborted all downstream stages. Fixed via sonar.exclusions as described above.

## Final successful run

All 13 stages green. Nexus upload confirmed. Docker image pushed to Docker Hub with build-number tag. Kubernetes deployment applied via RBAC-scoped token. Pods verified Running. App accessible at `http://<worker-node-ip>:31540`. Email notification sent.
