---
title: "PHASE 1B — Kubernetes Cluster: Node Setup (All 3 Nodes)"
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
order: 3
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01A-K8s-Cluster-EC2-Provisioning]]"
next: "[[Boardgame-01C-Kernel-Prerequisites]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1B — Kubernetes Cluster: Node Setup (All 3 Nodes)

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01A-K8s-Cluster-EC2-Provisioning|PHASE 1A — Kubernetes Cluster: EC2 Provisioning]] · → [[Boardgame-01C-Kernel-Prerequisites|PHASE 1C — Kernel Prerequisites (All 3 Nodes)]]
<!-- /nav -->

## Script created and run on all 3 nodes identically

```bash
cat > k8s-node-setup.sh << 'EOF'
#!/bin/bash
set -e

echo "=== 1. Update system packages ==="
sudo apt-get update

echo "=== 2. Install Docker ==="
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

echo "=== 3. Install cri-dockerd (CRI shim, required since K8s 1.24+ removed dockershim) ==="
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep '"tag_name"' | cut -d '"' -f4 | sed 's/^v//')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd_${VER}.3-0.ubuntu-jammy_amd64.deb
sudo dpkg -i cri-dockerd_${VER}.3-0.ubuntu-jammy_amd64.deb

echo "=== 4. Install Kubernetes apt dependencies ==="
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings

echo "=== 5. Add Kubernetes repo and GPG key ==="
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

echo "=== 6. Refresh package list ==="
sudo apt update

echo "=== 7. Install kubeadm, kubelet, kubectl (version-pinned) ==="
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
sudo apt-mark hold kubeadm kubelet kubectl

echo "✅ Node setup complete."
EOF
chmod +x k8s-node-setup.sh
./k8s-node-setup.sh
```

Why cri-dockerd is needed: Kubernetes removed `dockershim` (its built-in Docker integration) starting v1.24. Docker Engine alone is NOT CRI-compliant — kubelet can't talk to it directly anymore. `cri-dockerd` bridges that gap, implementing the CRI standard for Docker. Without it, `kubeadm init` fails its preflight check ("no valid CRI socket found"). The --cri-socket flag must be passed to every kubeadm init/join command.

Why apt-mark hold: prevents a routine `apt upgrade` from silently bumping kubeadm/kubelet/kubectl to a newer, potentially incompatible version — these three must stay version-matched with each other and with what the cluster was initialized with.

## Issue hit: MobaXterm session mix-up

When running on "slave-2," `hostname -I` actually showed slave-1's IP (172.31.6.84) — the MobaXterm saved session was duplicated from slave-1 without updating the IP. Fixed by creating a brand-new SSH session and verifying with `hostname -I` + `curl ifconfig.me` before running any commands. Lesson: always verify which server you're actually on before running join/init commands.
