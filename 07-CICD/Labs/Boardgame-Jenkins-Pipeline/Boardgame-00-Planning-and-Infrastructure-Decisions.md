---
title: "PHASE 0 — Planning & Infrastructure Decisions"
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
order: 1
up: "[[Boardgame-Jenkins-Pipeline-MOC]]"
next: "[[Boardgame-01A-K8s-Cluster-EC2-Provisioning]]"
origin: DevOps-Mastery/07-CICD/Labs/Boardgame Jenkins CICD Pipeline.md
---

# PHASE 0 — Planning & Infrastructure Decisions

<!-- nav -->
↑ [[Boardgame-Jenkins-Pipeline-MOC|Boardgame Jenkins CI/CD Pipeline — Complete Step-by-Step Notes]] · → [[Boardgame-01A-K8s-Cluster-EC2-Provisioning|PHASE 1A — Kubernetes Cluster: EC2 Provisioning]]
<!-- /nav -->

## Server topology decided

7 dedicated EC2 servers, deliberately separated rather than crammed onto one box:

|#|Server|Instance Type|Why this size|
|---|---|---|---|
|1|Jenkins + Docker + Trivy|m7i-flex.large (8GB/2vCPU)|Builds are memory-hungry; originally sized for all 3 tools combined, now dedicated to just Jenkins — generous headroom|
|2|Nexus|c7i-flex.large (4GB/2vCPU)|Nexus is lighter than SonarQube; 4GB is comfortable for a lab artifact repo|
|3|SonarQube + Postgres|m7i-flex.large (8GB/2vCPU)|SonarQube runs embedded Elasticsearch — genuinely RAM-hungry, wants 4GB+ realistically|
|4|K8s master|c7i-flex.large (4GB/2vCPU)|Runs etcd + API server + controller-manager + scheduler together; kubeadm's minimum is 2GB but that's too tight for real use|
|5|K8s worker 1 (slave-1)|t3.small (2GB/2vCPU)|Workers only run kubelet + kube-proxy + CNI + app pods — meets kubeadm minimum comfortably|
|6|K8s worker 2 (slave-2)|t3.small (2GB/2vCPU)|Same as slave-1|
|7|Monitoring (Prometheus + Grafana + Blackbox)|c7i-flex.large (4GB/2vCPU)|Prometheus's time-series DB grows with scrape targets; needs real headroom|

## Why separate servers, not all-in-one

Docker containers give process isolation but NOT resource isolation by default. Without explicit `--memory`/`--cpus` caps, every container shares the same CPU/RAM pool with zero enforced boundaries. SonarQube's embedded Elasticsearch commonly spikes to 2GB+ during scans and can starve Jenkins or Nexus via the OS OOM-killer. Real companies isolate these tools onto separate infrastructure.

## Why kubeadm, not EKS

Deliberate choice — already knew EKS from a previous project, wanted to understand what a managed control plane actually abstracts away (dockershim removal, CNI bootstrapping, kernel prerequisites, RBAC token generation) rather than only knowing the managed-service version.

## Key infrastructure choices made upfront

- All instances use Ubuntu 22.04 (Jammy) AMI
- All share the same security group (`myWebServer`, sg-09c50e5f8249e552e)
- All use the same SSH key pair (`myWebServer`)
- Region: ap-south-1 (Mumbai)
- All instance types verified as Free Tier eligible on this specific AWS account
