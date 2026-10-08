---
title: PHASE 1F — Security Auditing
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
order: 7
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
prev: "[[Boardgame-01E-Worker-Nodes-Join]]"
next: "[[Boardgame-02-Jenkins-Server-Setup]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 1F — Security Auditing

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · ← [[Boardgame-01E-Worker-Nodes-Join|PHASE 1E — Worker Nodes Join]] · → [[Boardgame-02-Jenkins-Server-Setup|PHASE 2 — Jenkins Server Setup]]
<!-- /nav -->

## kubeaudit (cluster configuration scanner)

```bash
VER=$(curl -s https://api.github.com/repos/Shopify/kubeaudit/releases/latest | grep '"tag_name"' | cut -d '"' -f4)
curl -L -o kubeaudit.tar.gz "https://github.com/Shopify/kubeaudit/releases/download/${VER}/kubeaudit_${VER#v}_linux_amd64.tar.gz"
tar -xvzf kubeaudit.tar.gz
sudo mv kubeaudit /usr/local/bin/
kubeaudit all
```

Findings grouped by actionability:

- Fixable: missing NetworkPolicies, missing resource limits, missing seccomp/AppArmor, runAsNonRoot not set
- Expected/not fixable: PrivilegedTrue + hostNetwork on Calico and kube-proxy (they genuinely need this to function — manipulating host iptables/network interfaces is their job)

kubeaudit's own banner flagged itself as deprecated (planned deprecation Oct 2024), recommending kube-bench instead.

## kube-bench (CIS Kubernetes Benchmark)

```bash
VER=$(curl -s https://api.github.com/repos/aquasecurity/kube-bench/releases/latest | grep '"tag_name"' | cut -d '"' -f4 | sed 's/^v//')
curl -L -o kube-bench.tar.gz "https://github.com/aquasecurity/kube-bench/releases/download/v${VER}/kube-bench_${VER}_linux_amd64.tar.gz"
tar -xvf kube-bench.tar.gz
sudo mv kube-bench /usr/local/bin/
```

First run had a subtle bug: running with `sudo` meant root's environment was used, which has no access to ~/.kube/config — kube-bench couldn't detect the real cluster version and silently fell back to testing against Kubernetes 1.18 controls (from 2020), producing a false PodSecurityPolicy finding for a feature removed in 1.25.

Corrected invocation:

```bash
sudo -E kube-bench run --targets master --version 1.28 --config-dir "$(pwd)/cfg" --config "$(pwd)/cfg/config.yaml"
```

Real results (against correct 1.28 benchmark): 38 PASS, 9 FAIL, 12 WARN.

Key FAILs identified:

- No audit logging configured at all (1.2.16-1.2.19)
- --profiling not disabled on API server, controller-manager, scheduler (1.2.15, 1.3.2, 1.4.1)
- --kubelet-certificate-authority not set (1.2.5)
- etcd data directory ownership (1.1.12)
