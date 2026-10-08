---
title: Compute Engine
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
order: 4
up: "[[GCP-MOC]]"
prev: "[[GCP-02-IAM-Service-Accounts-Roles]]"
next: "[[GCP-04-Persistent-Disk]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Compute Engine

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-02-IAM-Service-Accounts-Roles|IAM — Identity, Service Accounts, Roles]] · → [[GCP-04-Persistent-Disk|Persistent Disk]]
<!-- /nav -->

## Concept

Compute Engine is GCP's IaaS VM service. Rent a VM by the second, install what you want.

## AWS Equivalent

**Compute Engine = EC2**

## Azure Equivalent

**Compute Engine = Azure Virtual Machines**

Same category. Same use case. Different pricing model (sustained-use discounts, per-second billing), different machine type naming, different HA primitives.

## Key Differences from BOTH

**1. Per-Second Billing with 1-Minute Minimum**
AWS: per-second billing (with 1-min minimum for Linux, 1-hour for Windows historically).
Azure: per-second billing.
GCP: per-second billing, 1-minute minimum, applies to everyone.

Small difference — but GCP was first to per-second billing and it's aggressive about not rounding up.

**2. Sustained Use Discounts — AUTOMATIC**

GCP's killer pricing feature. Run a VM for more than 25% of the month, discounts kick in automatically. Up to 30% off if you run 24/7. **No commitment required.**

AWS and Azure require you to commit (Reserved Instances / Savings Plans) for equivalent discounts. GCP just gives it to you.

Committed Use Discounts (CUDs) exist too for deeper savings on 1-year or 3-year commits — those are the AWS RI / Azure Reserved Instance equivalents.

**3. Machine Type Naming**

| Family | Purpose | AWS Equivalent | Azure Equivalent |
|---|---|---|---|
| **E2** | General purpose, cost-optimized | AWS t3 | Azure B-series |
| **N2, N2D** | General purpose, balanced | AWS m5, m5a | Azure D-series |
| **C2, C2D** | Compute-optimized | AWS c5 | Azure F-series |
| **M2, M3** | Memory-optimized | AWS r5, x1 | Azure E-series |
| **A2, A3** | GPU (NVIDIA A100, H100) | AWS p4, p5 | Azure N-series |
| **T2A, T2D** | ARM (Ampere Altra, AMD Milan) | AWS Graviton | Azure Ampere |

Number roughly indicates generation, letter suffix indicates CPU vendor (D=AMD, A=ARM).

**4. Spot VMs and Preemptible VMs**

Older: **Preemptible VMs** — 24-hour max lifetime, evictable, ~80% off. Still supported for backward compatibility.
Newer: **Spot VMs** — no 24-hour limit, evictable, up to 91% off. **Use this for new workloads.**

Both AWS (Spot) and Azure (Spot VMs) have equivalents; GCP has both an older and newer version, which occasionally confuses documentation.

**5. Custom Machine Types**

Unique GCP feature: create VMs with any CPU/RAM combination you want. Need 3 vCPU with 15GB RAM? Just define it. AWS and Azure force you to pick from a catalog.

For most workloads, predefined types are fine and cheaper. Custom types are useful when you have precise memory-to-CPU ratios that don't match any predefined shape.

**6. Live Migration (No Reboots for Host Maintenance)**

GCP live-migrates VMs during host maintenance. Your VM keeps running, no reboot, no downtime. AWS and Azure use scheduled reboots for host maintenance.

## Real Configuration

```bash
# Enable API (first time in a project)
gcloud services enable compute.googleapis.com

# Create a VM
gcloud compute instances create vm-web-01 \
  --project=myapp-prod \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=web-server \
  --service-account=web-sa@myapp-prod.iam.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --metadata=startup-script='#!/bin/bash
    apt-get update && apt-get install -y nginx
    systemctl start nginx'

# List VMs
gcloud compute instances list

# SSH (gcloud handles keys automatically via OS Login or metadata-based keys)
gcloud compute ssh vm-web-01 --zone=us-central1-a

# Stop (billing continues for disk only — no compute charge; similar to Azure deallocate, DIFFERENT from AWS stop)
gcloud compute instances stop vm-web-01 --zone=us-central1-a

# Suspend (like laptop sleep — RAM saved to disk, faster resume, small charge for RAM disk)
gcloud compute instances suspend vm-web-01 --zone=us-central1-a

# Start
gcloud compute instances start vm-web-01 --zone=us-central1-a

# Delete (deletes attached disk by default unless --keep-disks specified)
gcloud compute instances delete vm-web-01 --zone=us-central1-a
```

## The Startup Script → user_data Equivalent

GCP metadata `startup-script` runs at boot — same idea as AWS user_data and Azure cloud-init:

```bash
gcloud compute instances create vm-web-01 \
  --metadata-from-file=startup-script=install.sh
```

## Production Use Case

**Standard web tier on Compute Engine:**

```
Project: myapp-prod-web
├── VPC: prod-vpc (custom-mode, subnet us-central1: 10.0.1.0/24)
├── Instance Template: web-tier-v1
│   ├── Machine: n2-standard-2 (2 vCPU, 8GB RAM)
│   ├── Image: cos-stable (Container-Optimized OS)
│   ├── Boot disk: pd-balanced 50GB
│   ├── Metadata: startup-script pulls app container and runs
│   ├── Service Account: web-sa (with cloud-platform scope)
│   └── Network tag: web-server (matches firewall rules)
├── Managed Instance Group: web-mig
│   ├── Regional MIG (spans 3 zones for HA)
│   ├── Autoscaler: 3-20 based on CPU > 60% for 3 min
│   └── Health check: HTTP /health on port 8080
└── HTTPS Load Balancer → web-mig backend
```

Deploy via Terraform. Instance Template is immutable — updates create a new template version and MIG rolls it out.

## Console Path

- Console → **Compute Engine** → **VM instances**
- Create: **CREATE INSTANCE** → wizard (machine type, boot disk, networking, service account)
- **Instance templates** — reusable VM specs
- **Instance groups** — MIGs (managed) and unmanaged groups
- Individual VM: **Details**, **Monitoring**, **Serial console** (like AWS EC2 serial console — helpful for debugging boot issues)

## Memory Hint

- **E2** = economical (cheapest, general-purpose)
- **N2** = normal (balanced default)
- **C2** = compute (CPU-heavy)
- **M2/M3** = memory (RAM-heavy)
- **Stop = free** in GCP (unlike Azure where you must deallocate)
- **Sustained Use Discounts are automatic** — this is the pricing feature to remember for interviews

## Interview Q&A — Compute Engine

**Q: Compare Compute Engine pricing to AWS EC2 and Azure VMs. What's the practical difference?**

> "Same on-demand rates roughly, but GCP has two pricing wins that matter in real workloads.
>
> First, Sustained Use Discounts. If you run a VM for a significant portion of the month, GCP automatically discounts it — up to 30% off for a 24/7 workload. Zero commitment. AWS and Azure require you to explicitly commit via Reserved Instances or Savings Plans for equivalent savings. This is a real financial edge for teams that don't want to make long-term commits but still run steady workloads.
>
> Second, per-second billing with 1-minute minimum applies across the board. AWS and Azure also have per-second billing now but with some edge cases. GCP is simpler.
>
> Third — the stop behavior. In AWS, stopping an EC2 instance immediately stops compute billing but you still pay for EBS. In Azure, you must explicitly deallocate to stop billing — a common gotcha. In GCP, stopping a VM stops compute billing immediately (disk still charged) — same behavior as AWS, cleaner than Azure.
>
> For interviews: the sustained use discount story is the easy win to mention. It shows you understand cloud economics beyond 'just pick the cheapest instance type.'"

**Q: When would you use Custom Machine Types instead of predefined?**

> "Custom Machine Types are worth reaching for when your workload has a specific CPU-to-memory ratio that doesn't fit predefined shapes well.
>
> Example: an in-memory cache that needs 4 vCPUs and 40GB of RAM. Predefined options:
> - `n2-standard-4` gives you 4 vCPU, 16GB RAM — too little memory
> - `n2-highmem-4` gives you 4 vCPU, 32GB RAM — still too little
> - `n2-highmem-8` gives you 8 vCPU, 64GB RAM — plenty of both but you're paying for CPUs you don't need
>
> With a custom machine type: 4 vCPU, 40GB RAM exactly. You pay for what you use, no waste.
>
> Trade-offs:
> - Custom types cost slightly more per unit than predefined (small premium)
> - You lose some pricing conveniences — Sustained Use Discounts still apply, but they're capped differently
> - Not all machine families support custom types (E2 does, but T2A/T2D are predefined-only)
>
> Practical rule: benchmark your workload first. If a predefined type wastes more than 20% of a resource type, consider custom. Otherwise, predefined is simpler.
>
> AWS and Azure don't have direct equivalents — you're locked to their catalogs. Some workloads move to GCP specifically for this flexibility (memory-heavy databases especially)."

**Q: An engineer complains their VM 'randomly reboots.' How do you respond?**

> "First question: is it actually a reboot, or is it a live migration? On GCP, live migration is the default for host maintenance — the VM keeps running, no downtime, but you might see a brief network blip.
>
> Check the operation history: `gcloud compute operations list --filter='targetLink:vm-web-01'`. Look for `compute.instances.migrateOnHostMaintenance` operations. If those match the timing, it's not a reboot — it's live migration doing its job.
>
> If it's an actual reboot, check the maintenance policy:
> - `TERMINATE` — VM will be shut down for maintenance (this is the setting for certain instance types like GPUs or Confidential VMs which can't be live-migrated)
> - `MIGRATE` — default, live migration
>
> If the policy is TERMINATE (usually not by choice — some instance families require it), maintenance events cause actual reboots. Solution: either accept it and design for stateless workloads, or migrate to an instance family that supports live migration.
>
> Other causes to check: kernel panic in serial console output, OOM killer, health check failures triggering auto-heal on a Managed Instance Group.
>
> Compare to AWS: no live migration. Every maintenance event = reboot with 2 weeks notice. Compare to Azure: also no live migration (host maintenance = reboot). Live migration is a real GCP differentiator for legacy workloads that hate downtime."
