---
title: PHASE 1D — Kubernetes Master Initialization
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
order: 5
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01C-Kernel-Prerequisites]]"
next: "[[Boardgame-01E-Worker-Nodes-Join]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1D — Kubernetes Master Initialization

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01C-Kernel-Prerequisites|PHASE 1C — Kernel Prerequisites (All 3 Nodes)]] · → [[Boardgame-01E-Worker-Nodes-Join|PHASE 1E — Worker Nodes Join]]
<!-- /nav -->

## kubeadm init on master only

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock
```

Why --pod-network-cidr: defines the IP range pods will use cluster-wide. 10.244.0.0/16 is the conventional default for Flannel-style CNI plugins; Calico also accepts it.

Why --cri-socket: explicitly tells kubeadm where to find the CRI endpoint — needed because Docker's default socket (/var/run/docker.sock) is NOT a CRI socket; cri-dockerd exposes its own at /var/run/cri-dockerd.sock.

## Configure kubectl access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Why: kubeadm generates a cluster-admin kubeconfig at /etc/kubernetes/admin.conf (root-owned). Copying it to your home directory with correct ownership lets your regular user run kubectl without sudo.

## Deploy Calico CNI

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Why Calico specifically: it's the CNI plugin this course uses. Handles pod IP assignment, inter-node pod routing, and network policy enforcement. Without a CNI plugin, nodes stay NotReady and pods can't communicate.

## Deploy nginx Ingress Controller — hit a version-mismatch bug

Original command from the course docs:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```

This caused a CrashLoopBackOff — diagnosed via:

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50
```

Root cause found in logs:

```
Failed to watch *v1beta1.Ingress: failed to list *v1beta1.Ingress: the server could not find the requested resource
```

The controller version (v0.49.0, from ~2021) was hardcoded to use the `networking.k8s.io/v1beta1` Ingress API, which was fully removed in Kubernetes 1.22 — our cluster runs 1.28, six versions past removal.

Fixed with a current version:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml
```
