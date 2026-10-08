---
title: PHASE 1C — Kernel Prerequisites (All 3 Nodes)
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
order: 4
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01B-K8s-Node-Setup]]"
next: "[[Boardgame-01D-K8s-Master-Initialization]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1C — Kernel Prerequisites (All 3 Nodes)

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01B-K8s-Node-Setup|PHASE 1B — Kubernetes Cluster: Node Setup (All 3 Nodes)]] · → [[Boardgame-01D-K8s-Master-Initialization|PHASE 1D — Kubernetes Master Initialization]]
<!-- /nav -->

## Initially skipped these steps — then hit the exact failure they prevent

```
error execution phase preflight: [preflight] Some fatal errors occurred:
    [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
```

## Applied on all 3 nodes

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Why overlay: the overlay filesystem driver — container runtimes (containerd/Docker) use this for efficient, layered container image storage.

Why br_netfilter: allows iptables to see and filter bridged network traffic. Kubernetes' pod networking relies on bridge networking + iptables rules for service routing; without this module, pod-to-pod traffic across the bridge wouldn't be visible to netfilter, breaking Service routing entirely.

Why ip_forward: enables the node to forward IP packets between interfaces — essential since a K8s node routes traffic between pods, and between pods and the outside world.

Why these must be on ALL 3 nodes, not just master: every node runs pod networking, not just the control plane.
