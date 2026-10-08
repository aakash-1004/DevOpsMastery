---
title: Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes
aliases:
  - Boardgame Jenkins CICD Pipeline
  - Boardgame-Jenkins-CICD-Full-Project
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
date: 2026-08-25 to 2026-08-28
up: "[[CICD-MOC]]"
related:
  - "[[7.4.1-Jenkins-Controller-Agent-Architecture]]"
  - "[[7.4.2-Jenkins-Declarative-Pipeline-Anatomy]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes

<!-- nav -->
↑ [[CICD-MOC|CI/CD]]
<!-- /nav -->

Everything that was done, in chronological order, no step omitted regardless of how small.

## Contents

- [[Boardgame-00-Planning-and-Infrastructure-Decisions|PHASE 0 — Planning & Infrastructure Decisions]] — Server topology decided; Why separate servers, not all-in-one; Why kubeadm, not EKS; Key infrastructure choices made upfront
- [[Boardgame-01A-K8s-Cluster-EC2-Provisioning|PHASE 1A — Kubernetes Cluster: EC2 Provisioning]] — Launched 3 instances for the K8s cluster; Added self-referencing security group rule for inter-node traffic; Fetched all instance IPs
- [[Boardgame-01B-K8s-Node-Setup|PHASE 1B — Kubernetes Cluster: Node Setup (All 3 Nodes)]] — Script created and run on all 3 nodes identically; Issue hit: MobaXterm session mix-up
- [[Boardgame-01C-Kernel-Prerequisites|PHASE 1C — Kernel Prerequisites (All 3 Nodes)]] — Initially skipped these steps — then hit the exact failure they prevent; Applied on all 3 nodes
- [[Boardgame-01D-K8s-Master-Initialization|PHASE 1D — Kubernetes Master Initialization]] — kubeadm init on master only; Configure kubectl access; Deploy Calico CNI; Deploy nginx Ingress Controller — hit a version-mismatch bug
- [[Boardgame-01E-Worker-Nodes-Join|PHASE 1E — Worker Nodes Join]] — Generate join command on master; Run on each worker (slave-1 and slave-2), with --cri-socket appended; Verified on master; Note: kubectl only works on the master
- [[Boardgame-01F-Security-Auditing|PHASE 1F — Security Auditing]] — kubeaudit (cluster configuration scanner); kube-bench (CIS Kubernetes Benchmark)
- [[Boardgame-02-Jenkins-Server-Setup|PHASE 2 — Jenkins Server Setup]] — Jenkins instance was already provisioned earlier (m7i-flex.large); Full setup script (final corrected version, incorporating all bugs fixed during the session); Bugs hit and fixed during Jenkins setup; Install kubectl on Jenkins server; Jenkins initial configuration (via browser); Jenkins Global Tool Configuration (Manage Jenkins → Tools)
- [[Boardgame-03-Nexus-Server-Setup|PHASE 3 — Nexus Server Setup]] — Launched instance; Setup script; Get initial admin password; Nexus initial configuration (via browser at :8081)
- [[Boardgame-04-SonarQube-and-Postgres-Server-Setup|PHASE 4 — SonarQube + Postgres Server Setup]] — Launched instance; Setup script; SonarQube initial configuration (via browser at :9000)
- [[Boardgame-05-Source-Code-Private-Repo-Setup|PHASE 5 — Source Code: Private Repo Setup]] — Cloned reference app, stripped history, pushed to own private repo; Modified pom.xml
- [[Boardgame-06-RBAC-Jenkins-to-Kubernetes-Access|PHASE 6 — RBAC: Jenkins → Kubernetes Deployment Access]] — Create namespace; Create ServiceAccount; Create Role (namespace-scoped, not ClusterRole); Create RoleBinding; Create token Secret (required since K8s 1.24); Retrieve token and add to Jenkins
- [[Boardgame-07-Jenkinsfile-Writing-Debugging-Correction|PHASE 7 — Jenkinsfile: Writing, Debugging, and Correction]] — Initial Jenkinsfile written by hand (from the video), with bugs caught in review; deployment-service.yaml — 3 bugs caught before it could fail silently; BUILD_TAG_PLACEHOLDER mechanism; Final working Jenkinsfile (13 stages + post block); sonar.exclusions — a bug traced to a single missing character; Pipeline-from-SCM conversion
- [[Boardgame-08-Pipeline-Runs-and-Real-Debugging|PHASE 8 — Pipeline Runs and Real Debugging]] — First real run — 8 stages passed, failed at Docker build; Second run — Quality Gate failure (genuine enforcement); Final successful run
- [[Boardgame-09-Monitoring-Stack|PHASE 9 — Monitoring Stack]] — Launched monitoring server; Prometheus installation (step by step, raw binary — not apt); YAML config bug hit twice; Node Exporter installation (on monitoring server, then on Jenkins server); Grafana installation (via apt — auto-creates systemd service); Blackbox Exporter installation
- [[Boardgame-10-Repo-Polish|PHASE 10 — Repo Polish]] — Added RBAC manifests to the repo; Updated README.md; Added .dockerignore; Updated .gitignore
- [[Boardgame-11-Instance-IDs-Reference|11. Instance IDs — Full Reference]] — Stop all instances (to pause billing)

<!-- related -->
---
**Related:** [[7.4.1-Jenkins-Controller-Agent-Architecture|Jenkins — Controller/Agent Architecture]] · [[7.4.2-Jenkins-Declarative-Pipeline-Anatomy|Jenkins — Declarative Pipeline Anatomy]]
<!-- /related -->
