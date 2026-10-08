---
title: EKS — Elastic Kubernetes Service
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 17
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-16-ECS]]"
next: "[[AWS-18-Amplify]]"
related:
  - "[[5.5-EKS-Fargate-IRSA-ALB-Project]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# EKS — Elastic Kubernetes Service

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-16-ECS|ECS — Elastic Container Service]] · → [[AWS-18-Amplify|Amplify — Fullstack Platform]]
<!-- /nav -->

## Concept

**AWS EKS (Elastic Kubernetes Service)** is a managed Kubernetes service. AWS runs the control plane, you manage worker nodes (or use Fargate for serverless nodes).

**Why managed K8s:**

- Running your own K8s control plane is complex (etcd, API server, scheduler, controllers)
- Requires HA setup across AZs
- Requires patching, upgrades, security hardening
- EKS handles all of this — you focus on your workloads

## Kubernetes Refresher

Kubernetes is an open-source platform for automating deployment, scaling, and management of containerized applications.

**Cluster components:**

```
Control Plane (managed by AWS in EKS)
    ├── kube-apiserver (brain, all comms)
    ├── etcd (state storage)
    ├── kube-scheduler (places pods on nodes)
    └── kube-controller-manager (runs controllers)

Worker Nodes (you manage — or Fargate)
    ├── kubelet (node agent)
    ├── kube-proxy (networking)
    ├── containerd (runtime)
    └── Your pods running here
```

## EKS Node Options

```
1. Self-Managed EC2 Nodes
   ├── You launch and manage EC2 instances
   ├── Add to cluster via user_data or eksctl
   └── Full control, more work

2. Managed Node Groups
   ├── AWS manages EC2 lifecycle (launch, terminate, upgrade)
   ├── Easier to operate
   ├── Auto Scaling built-in
   └── Recommended for most use cases

3. Fargate
   ├── Serverless nodes — no EC2 management
   ├── Pay per pod (vCPU + memory + time)
   ├── Not all K8s features supported (DaemonSets, some networking)
   └── Best for stateless, variable workloads
```

## Prerequisites — CLI Tools

**AWS CLI:**

```bash
# Verify installation
aws --version

# Configure credentials
aws configure

# Verify identity
aws sts get-caller-identity
```

**eksctl** (EKS-specific CLI, easiest way to create clusters):

```bash
# Windows (Chocolatey)
choco install eksctl

# Mac (Homebrew)
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Verify
eksctl version
```

**kubectl** (Kubernetes CLI):

```bash
# Mac
brew install kubectl

# Windows
choco install kubernetes-cli

# Verify
kubectl version --client
```

## Creating an EKS Cluster

**Simple cluster with AutoMode:**

```bash
eksctl create cluster --name=my-cluster --enable-auto-mode
```

**With more options:**

```bash
eksctl create cluster \
  --name=production-cluster \
  --region=ap-south-1 \
  --node-type=t3.medium \
  --nodes=3 \
  --nodes-min=2 \
  --nodes-max=5 \
  --managed
```

**Options explained:**

- `--nodes=3` — start with 3 worker nodes
- `--nodes-min=2, --nodes-max=5` — Auto Scaling range
- `--node-type=t3.medium` — EC2 instance type
- `--managed` — managed node group (recommended)
- `--fargate` — use Fargate instead of EC2

**Takes 15-20 minutes** to fully provision. AWS creates VPC, subnets, IAM roles, control plane, node group.

## After Cluster Creation

```bash
# Update kubectl config to point to EKS cluster
aws eks update-kubeconfig --name my-cluster --region ap-south-1

# Verify cluster access
kubectl get nodes

# Deploy your app
kubectl apply -f deployment.yaml
```

## EKS Architecture

```
AWS Managed Control Plane
    ├── Multi-AZ HA (across 3 AZs)
    ├── Auto-patched, auto-upgraded
    ├── Costs $0.10/hour ($73/month) per cluster
    └── Accessed via kubectl

Worker Nodes (in your VPC)
    ├── EC2 or Fargate
    ├── Deployed across multiple AZs
    ├── Can use Cluster Autoscaler for automatic scaling
    └── You pay for these instances

Networking
    ├── AWS VPC CNI plugin (default)
    ├── Each pod gets a real VPC IP
    ├── Direct integration with Security Groups
    └── High performance networking

Storage
    ├── EBS CSI driver for persistent volumes
    ├── EFS CSI driver for shared storage
    └── S3 for object storage
```

## Real Production Setup

```
EKS Cluster (production)
    ├── Managed Node Group 1: m5.large × 3 (production workloads)
    ├── Managed Node Group 2: c5.xlarge × 2 (CPU-intensive services)
    ├── Fargate Profile: system pods (kube-system namespace)
    │
    ├── Deployed apps:
    │   ├── Namespace: production
    │   │   ├── Deployment: api-service (5 replicas)
    │   │   ├── Deployment: worker (3 replicas)
    │   │   └── HPA scaling on CPU
    │   ├── Namespace: monitoring
    │   │   ├── Prometheus
    │   │   └── Grafana
    │   └── Namespace: ingress-nginx
    │       └── Ingress controller (routes external traffic)
    │
    ├── Cluster Autoscaler (scales node groups based on pod demand)
    ├── Metrics Server (for HPA)
    ├── AWS Load Balancer Controller (for Ingress → ALB)
    └── External DNS (auto-creates Route 53 records)
```

## Concepts to Master for EKS

Deep Kubernetes knowledge (revisit K8s notes):

- **Pods** — smallest unit
- **Deployments** — manages replicas, rolling updates
- **Services** — internal networking, load balancing
- **Ingress** — external HTTP/HTTPS routing
- **ConfigMaps & Secrets** — configuration and credentials
- **RBAC** — access control
- **HPA** — horizontal pod autoscaling
- **StatefulSets** — for databases, stateful apps
- **PersistentVolumes** — storage

## Memory Hints

- **EKS** = "AWS-managed Kubernetes"
- **eksctl** = "easy EKS cluster creator"
- **Managed Node Group** = "AWS-managed worker nodes"
- **Fargate on EKS** = "serverless K8s nodes"

## Interview Q&A — EKS

**Q: What are the advantages of managed EKS over self-managed K8s on EC2?**

> "Three main advantages: reliability, security, and operational efficiency.
>
> Reliability: EKS runs the control plane across multiple AZs automatically. You get 99.95% SLA. Self-managed means you build and maintain HA control plane yourself — hard to get right, requires ongoing expertise.
>
> Security: AWS patches the control plane automatically. Handles CVE responses for etcd, kube-apiserver, and other components. You focus on your workloads.
>
> Operational efficiency: control plane upgrades are much easier — you initiate an upgrade, AWS handles the multi-step process. Self-managed upgrades are notoriously complex.
>
> Cost is the trade-off — $73/month per cluster. But for anything production, that's negligible compared to the engineering time saved. And AWS handles disaster recovery, backup of etcd, and other complex operations.
>
> My recommendation: unless you have a compliance requirement forcing on-prem or specific control plane customization, use EKS. The team's time is better spent on application-level work than K8s control plane maintenance."

**Q: What's the difference between EKS with EC2 and EKS with Fargate?**

> "Different node models.
>
> EKS with EC2 (Managed Node Groups): AWS provisions and manages EC2 instances that run your pods. You choose instance types, size the node group. You have visibility and control over the nodes. Can SSH in (though shouldn't need to). Pay for EC2 instances 24/7.
>
> EKS with Fargate: Fully serverless — no EC2 to manage. AWS runs pods on hidden infrastructure. You define Fargate profiles specifying which pods run on Fargate (by namespace, labels). Pay per pod (vCPU + memory + time). No node management, no capacity planning.
>
> Fargate limitations: no DaemonSets, no privileged containers, no HostNetwork, no HostPath volumes. Not suitable for monitoring agents that need node-level access.
>
> Common pattern: mix both. Fargate for stateless apps that scale up/down frequently. EC2 nodes for system pods (monitoring, ingress controllers, DaemonSets) that need node access. Best of both worlds."

<!-- related -->
---
**Related:** [[5.5-EKS-Fargate-IRSA-ALB-Project|EKS on Fargate + IRSA + ALB Ingress Controller — End to End]]
<!-- /related -->
