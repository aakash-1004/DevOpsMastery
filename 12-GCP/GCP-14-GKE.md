---
title: GKE — Google Kubernetes Engine
type: reference
module: gcp
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - gcp
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
order: 15
up: "[[GCP-MOC]]"
prev: "[[GCP-13-Cloud-CDN]]"
next: "[[GCP-15-VPC-Networking]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# GKE — Google Kubernetes Engine

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-13-Cloud-CDN|Cloud CDN]] · → [[GCP-15-VPC-Networking|VPC Networking]]
<!-- /nav -->

> **Tags:** #gcp #gke #kubernetes #core
> **Status:** Critical — this is the deepest section for a reason. Kubernetes was invented at Google. GKE is arguably the most mature managed K8s offering, and it's the strongest cross-cloud interview material for someone with a Kubernetes background.
> **Interview Relevance:** ⭐⭐⭐⭐⭐

## Concept

**GKE (Google Kubernetes Engine)** is GCP's managed Kubernetes service. Google open-sourced Kubernetes in 2014 based on their internal Borg system, so GKE has a philosophical claim to being "K8s done right" — features often land here before EKS or AKS.

Two operating modes:
- **Standard mode** — you manage node pools, node sizing, upgrades. Full control. Like EKS or AKS.
- **Autopilot mode** — Google manages nodes entirely. You define pods, Google provisions the exact node capacity needed, per-pod billing. No node concept for you as a user. Closest analog: Fargate for EKS, but more integrated and cheaper for spiky workloads.

## AWS Equivalent

- **EKS (Elastic Kubernetes Service)** — managed control plane, worker nodes you manage (or Fargate for serverless).

## Azure Equivalent

- **AKS (Azure Kubernetes Service)** — managed control plane (free), you manage nodes.

## Key Differences from BOTH

| Aspect | AWS EKS | Azure AKS | GCP GKE |
|---|---|---|---|
| Control plane cost | ~$73/mo per cluster | Free | Standard: ~$73/mo (first cluster free per billing account); Autopilot: no separate charge |
| Node management | You manage OR Fargate | You manage | You manage OR Autopilot |
| Serverless pods | Fargate (limited features) | Virtual Nodes (ACI-backed) | **Autopilot (fully featured, no compromise)** |
| Kubernetes version freshness | Follows upstream, slower | Follows upstream | **Often first to release new versions** |
| Networking | VPC CNI (assigns VPC IPs to pods) | Azure CNI or kubenet | **VPC-native (alias IPs) — pods get real VPC IPs by default** |
| Workload identity | IRSA (IAM Roles for Service Accounts) | Workload Identity (via Entra) | **Workload Identity (SA-to-SA mapping — cleanest of the three)** |
| Auto-upgrade | Manual by default | Manual by default | **Auto-upgrade enabled by default** on release channels |
| DNS | CoreDNS | CoreDNS | CoreDNS + kube-dns (Google's fork, being deprecated in favor of CoreDNS) |
| Ingress default | ALB Controller (add-on) | App Gateway Ingress Controller | **GKE Ingress (built-in, provisions Global External LB)** |

**The big differences to lead with in interviews:**

1. **Autopilot** — no other cloud has an equivalent that's this seamless. Fargate has limitations (no DaemonSets, no privileged pods, no host networking); Autopilot supports almost everything Standard supports. You pay per pod's CPU/memory/ephemeral-storage request, second-billed. For spiky or unpredictable workloads, it's dramatically cheaper than provisioning nodes.
2. **Release channels** — Rapid / Regular / Stable channels for automatic upgrades. Set once, forget. EKS and AKS require manual intervention.
3. **VPC-native networking by default** — pods get IPs from a dedicated subnet range, routable across the VPC, no overlay. Simplifies debugging and hybrid connectivity massively.
4. **Workload Identity is the cleanest** — bind a Kubernetes ServiceAccount to a GCP ServiceAccount via annotation. Pods automatically get GCP creds. No sidecars, no init containers, no token brokers.

## Real Configuration

```bash
# ==========================================
# STANDARD MODE — full control
# ==========================================

# Create a Standard cluster
gcloud container clusters create prod-cluster \
  --region=asia-south1 \
  --release-channel=regular \
  --enable-ip-alias \
  --network=prod-vpc \
  --subnetwork=prod-subnet-asia \
  --cluster-secondary-range-name=pods-range \
  --services-secondary-range-name=services-range \
  --workload-pool=my-project-956179.svc.id.goog \
  --enable-shielded-nodes \
  --shielded-secure-boot \
  --enable-master-authorized-networks \
  --master-authorized-networks=10.0.0.0/24 \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --num-nodes=1 \
  --machine-type=e2-medium

# Flags explained:
#   --release-channel=regular     : auto-upgrade on regular channel (stable is more conservative)
#   --enable-ip-alias             : VPC-native (pods get real VPC IPs). ALWAYS set this.
#   --workload-pool=...           : enables Workload Identity Federation
#   --enable-private-nodes        : nodes have no public IPs
#   --master-ipv4-cidr            : control plane's private CIDR (must be /28)
#   --enable-shielded-nodes       : verified boot, integrity monitoring

# Add a second node pool (e.g., high-memory for a specific workload)
gcloud container node-pools create memory-pool \
  --cluster=prod-cluster \
  --region=asia-south1 \
  --machine-type=n2-highmem-4 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --node-taints=workload=memory-intensive:NoSchedule \
  --node-labels=pool=memory

# Add a Spot node pool for batch jobs (up to 91% cheaper)
gcloud container node-pools create spot-pool \
  --cluster=prod-cluster \
  --region=asia-south1 \
  --machine-type=e2-standard-4 \
  --spot \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=10 \
  --node-taints=cloud.google.com/gke-spot=true:NoSchedule

# Get kubeconfig
gcloud container clusters get-credentials prod-cluster --region=asia-south1

# ==========================================
# AUTOPILOT MODE — fully managed pods
# ==========================================

gcloud container clusters create-auto autopilot-cluster \
  --region=asia-south1 \
  --release-channel=regular \
  --network=prod-vpc \
  --subnetwork=prod-subnet-asia

# That's it. No node config. No autoscaler. No node pool decisions.
# Deploy a pod, Google provisions capacity. Pay per pod's resource requests.

# ==========================================
# WORKLOAD IDENTITY — pod-to-GCP-service auth
# ==========================================

# 1. Create GCP service account
gcloud iam service-accounts create app-gsa \
  --display-name="App workload identity"

# 2. Grant it a role (e.g., read from Cloud Storage)
gcloud projects add-iam-policy-binding my-project-956179 \
  --member="serviceAccount:app-gsa@my-project-956179.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# 3. Create Kubernetes ServiceAccount
kubectl create serviceaccount app-ksa -n production

# 4. Bind them (this is the magic)
gcloud iam service-accounts add-iam-policy-binding \
  app-gsa@my-project-956179.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project-956179.svc.id.goog[production/app-ksa]"

# 5. Annotate the K8s SA
kubectl annotate serviceaccount app-ksa -n production \
  iam.gke.io/gcp-service-account=app-gsa@my-project-956179.iam.gserviceaccount.com

# 6. In your deployment spec, use serviceAccountName: app-ksa
# Pods now authenticate to GCP as app-gsa. Zero secrets, zero sidecars.
```

## Production Use Case

**Scenario:** A team is running a Node.js + Postgres app on GKE Standard. Traffic is highly variable — heavy business hours, near-zero at night. They're overpaying because nodes are provisioned for peak load.

**Approach (senior-level answer):**

> "First, break the workload apart by traffic pattern:
> - Steady, predictable pods (auth service, background workers) → main node pool, e2-standard-2 with autoscaling min=2 max=6, 1-year committed use discount on the min. Locks in ~37% off for the baseline you're always running.
> - Bursty pods (main app) → **migrate to Autopilot cluster** or add an Autopilot node pool if we want to stay in Standard. Pay only for what pods actually request during the burst; scale to zero at night.
> - Batch/analytics pods (report generation, data pipelines) → dedicated **Spot node pool** with taints, so only tolerating pods land there. Up to 91% cheaper. Add PodDisruptionBudgets so if Spot nodes get preempted, we don't lose too many replicas at once.
>
> Then Workload Identity for every namespace's pods — no more static secrets in Kubernetes secrets for GCP API access. Cloud SQL Auth Proxy runs as a sidecar or a separate deployment, authenticating via Workload Identity to the Cloud SQL instance — no DB passwords in K8s at all.
>
> For rollouts: enable release channel `regular`, set maintenance windows to align with the app's lowest-traffic hours. GKE handles control plane and node upgrades automatically.
>
> Cost outcome: typically 40-60% reduction vs a fixed-size Standard cluster once you factor in Spot + Autopilot + committed use."

## Console Path

- Console → **Kubernetes Engine** → **Clusters**
- **Workloads** for pod-level view (integrated with cluster)
- **Services & Ingress** for LB integration

## Memory Hint

- **GKE Standard** = "EKS/AKS-style: you manage nodes"
- **GKE Autopilot** = "Fargate but it actually works for real workloads"
- **Workload Identity** = "IRSA but simpler — just annotate the ServiceAccount"
- **Release channels** = "Rapid = bleeding edge, Regular = normal, Stable = conservative"
- **Always `--enable-ip-alias`** — VPC-native is the default modern choice

## Interview Q&A — GKE

**Q: Walk me through how you'd choose between GKE Standard and Autopilot for a new project.**

> "It comes down to three factors: workload predictability, cost model, and control needs.
>
> Autopilot wins when:
> - Traffic is variable or unpredictable — you pay per-pod resource request, per-second billed, so idle capacity costs nothing
> - Team wants to focus on apps, not infrastructure — no node sizing, no autoscaler tuning, no OS patching
> - Workloads are standard — HTTP services, workers, jobs
>
> Standard wins when:
> - You need specific machine types — GPUs, high-memory, ARM (T2A)
> - Workloads require host networking, privileged pods, DaemonSets across all nodes, or custom kernel modules — Autopilot restricts these
> - You have steady baseline load — committed use discounts on Standard nodes beat Autopilot's per-pod pricing
> - You want fine control over networking, kubelet config, or custom node images
>
> My default recommendation for a new project: **start with Autopilot**. If it hits a limitation you actually need, migrate specific workloads to a Standard cluster. Most teams over-optimize for control they never end up using."

**Q: Explain Workload Identity in GKE — how it compares to IRSA in EKS and Azure AD Workload Identity in AKS.**

> "All three solve the same problem: giving Kubernetes pods cloud-native credentials without embedding secrets. But the mechanisms differ meaningfully.
>
> **EKS IRSA** — you associate an IAM Role with a Kubernetes ServiceAccount via an OIDC provider. Pods running under that SA get temporary AWS credentials via the AWS SDK's default credential chain. Requires setting up an OIDC identity provider in IAM, per cluster.
>
> **AKS Workload Identity** — federated credential in Entra maps a Kubernetes SA to an Entra managed identity. Pods get Entra tokens via projected volume. Newer feature — replaced the older 'aad-pod-identity' which had reliability issues.
>
> **GKE Workload Identity** — Kubernetes ServiceAccount is bound to a GCP ServiceAccount via IAM binding, plus an annotation on the KSA. That's it. Google runs an identity proxy on every node that intercepts metadata server requests from pods and returns the mapped GSA's tokens.
>
> Why GKE's is cleanest: no separate OIDC provider setup per cluster (workload-pool is a single project-level setting), no sidecar or projected volume config, and the mapping is bidirectional-explicit — the GSA has to explicitly allow the KSA to impersonate it. Very hard to accidentally over-grant."

**Q: Kubernetes was invented at Google. Does that actually matter for GKE, or is it marketing?**

> "It's partly historical, partly real. The historical part: Borg predates Kubernetes by a decade, and many concepts — pods, controllers, declarative reconciliation — are Borg-derived. GKE inherits operational patterns Google refined internally for years.
>
> The real part: GKE often ships upstream K8s features first. Autopilot didn't exist elsewhere for years. Workload Identity was mature on GKE before AKS shipped its equivalent. GKE's control plane is genuinely more robust — I've seen fewer control plane incidents on GKE than EKS in comparable production usage.
>
> That said, EKS has caught up substantially in the last two years — Karpenter, EKS Auto Mode, Fargate improvements. AKS has closed the gap too. It's not a landslide anymore. But if I were starting a Kubernetes-heavy project on a clean slate, GKE Autopilot on the Regular release channel is still my default recommendation."
