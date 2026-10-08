---
title: PHASE 1E — Worker Nodes Join
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
order: 6
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01D-K8s-Master-Initialization]]"
next: "[[Boardgame-01F-Security-Auditing]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1E — Worker Nodes Join

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01D-K8s-Master-Initialization|PHASE 1D — Kubernetes Master Initialization]] · → [[Boardgame-01F-Security-Auditing|PHASE 1F — Security Auditing]]
<!-- /nav -->

## Generate join command on master

```bash
kubeadm token create --print-join-command
```

## Run on each worker (slave-1 and slave-2), with --cri-socket appended

```bash
sudo kubeadm join 172.31.33.140:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///var/run/cri-dockerd.sock
```

Why --cri-socket is needed here too: same reason as master init — workers also need to know where the CRI endpoint is, since Docker itself doesn't expose one.

## Verified on master

```bash
kubectl get nodes
```

Result: all 3 nodes Ready (master = control-plane role, workers = <none> role).

## Note: kubectl only works on the master

Running kubectl on a worker gives "connection refused" — workers don't have the kubeconfig file pointing at the API server. kubectl is a client tool for the API server, which only runs on the control plane. Normal, expected architecture.
