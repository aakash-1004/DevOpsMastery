---
title: MIG — Managed Instance Groups
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
order: 8
up: "[[GCP-MOC]]"
prev: "[[GCP-06-Cloud-Load-Balancing]]"
next: "[[GCP-08-Cloud-SQL]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# MIG — Managed Instance Groups

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-06-Cloud-Load-Balancing|Cloud Load Balancing]] · → [[GCP-08-Cloud-SQL|Cloud SQL]]
<!-- /nav -->

## Concept

Managed Instance Group (MIG) is GCP's autoscaling group primitive — a fleet of identical VMs from a template, with autoscaling, autohealing, and rolling updates.

## AWS Equivalent

**MIG = AWS Auto Scaling Group + Launch Template combined**

## Azure Equivalent

**MIG = Azure Virtual Machine Scale Set (VMSS)**

## Three-Way Comparison

| Feature | AWS ASG | Azure VMSS | GCP MIG |
|---|---|---|---|
| Template resource | Launch Template | (embedded in VMSS) | Instance Template |
| Scaling policies | Target Tracking + Step + Scheduled | Autoscale profiles + Application Insights metrics | Autoscaler + custom metrics |
| Health check | ELB or EC2 | Application Health Extension | HTTP/TCP/SSL health check |
| Zone spread | Multi-AZ via subnet mix | Zones flag | Zonal MIG or Regional MIG |
| Rolling update | Instance Refresh | Rolling upgrade | Rolling update (built-in) |
| Spot integration | Mixed instance policy | Spot priority in pool | Spot VMs in MIG |

## Key Differences from BOTH

**1. Regional vs Zonal MIGs**

Two flavors:
- **Zonal MIG** — all instances in one zone. Simpler, no HA.
- **Regional MIG** — instances spread across 3 zones automatically. HA by default.

**Always use Regional MIGs for production.** The overhead is minimal.

AWS: ASG can span multiple AZs via subnet configuration.
Azure: VMSS with `--zones 1 2 3` spans zones.
GCP: it's a first-class Regional MIG option — cleaner.

**2. Autohealing**

If a VM fails a health check, MIG kills and replaces it. Configurable initial delay to allow startup time.

Same feature exists in AWS ASG and Azure VMSS. GCP's is slightly cleaner to configure — one health check spec used for both LB health and autohealing.

**3. Instance Templates Are Versioned**

You create a new Instance Template for changes, then set the MIG to use the new version. Rolling updates handle the transition automatically. Old template stays around for rollback.

AWS Launch Templates are versioned similarly. Azure VMSS uses "model" versioning.

## Real Configuration

```bash
# 1. Create an instance template
gcloud compute instance-templates create web-template-v1 \
  --machine-type=n2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=web-server \
  --service-account=web-sa@myapp-prod.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata=startup-script='#!/bin/bash
    apt-get update && apt-get install -y nginx
    systemctl start nginx'

# 2. Create a health check
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/ \
  --check-interval=10s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# 3. Create Regional MIG
gcloud compute instance-groups managed create web-mig \
  --template=web-template-v1 \
  --size=3 \
  --region=us-central1 \
  --health-check=web-health-check \
  --initial-delay=180

# 4. Set autoscaling
gcloud compute instance-groups managed set-autoscaling web-mig \
  --region=us-central1 \
  --min-num-replicas=3 \
  --max-num-replicas=20 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90

# 5. Rolling update to new template
gcloud compute instance-templates create web-template-v2 \
  --machine-type=n2-standard-4 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --service-account=web-sa@myapp-prod.iam.gserviceaccount.com \
  --scopes=cloud-platform

gcloud compute instance-groups managed rolling-action start-update web-mig \
  --version=template=web-template-v2 \
  --region=us-central1 \
  --max-surge=3 \
  --max-unavailable=0
```

## Production Use Case

**Auto-scaling web tier with mixed on-demand + Spot:**

```
Regional MIG: web-mig (us-central1, 3 zones)
├── Instance template v3 (n2-standard-4, cos-stable image, container-based app)
├── Autoscaling:
│   ├── Target CPU: 60%
│   ├── Min: 3, Max: 30
│   ├── Cool-down: 90s
│   └── Custom metric: HTTP requests/sec > 200 → scale out
├── Health check: HTTP /health on 8080, 10s interval
├── Autoheal: if 3 consecutive failures, replace VM
├── Update policy: rolling, max-surge 3, max-unavailable 0 (no downtime)
└── Frontend: attached to Global External Application LB as backend
```

## Console Path

- Console → **Compute Engine** → **Instance groups**
- Create MIG wizard → template, region, size, autoscaling policy, health check
- Existing MIG: **Details** → instance list, autoscaling, autohealing, updates history

## Memory Hint

- **MIG = ASG + Launch Template combined** (like VMSS)
- **Regional MIG** = HA default (3 zones), always use for prod
- **Zonal MIG** = single zone (dev/test only)
- **Instance Template** = versioned VM spec
- **Rolling update** built-in — no separate deploy tool needed

## Interview Q&A — MIG

**Q: Compare AWS ASG, Azure VMSS, and GCP MIG. Which is easiest to operate?**

> "Same category, similar features, subtle usability differences.
>
> Configuration overhead: MIG and VMSS both bundle template + scaling into one resource. AWS ASG separates Launch Template from ASG which is more composable but more moving parts to manage.
>
> Rolling updates: GCP MIG has rolling updates as a first-class operation with `rolling-action start-update`. Very clean. AWS ASG has Instance Refresh — also good but slightly more configuration. Azure VMSS has rolling upgrades but the mental model is a bit different.
>
> Multi-zone spread: Regional MIG is a checkbox — 'spread across zones.' AWS ASG requires you to specify subnets across AZs. Azure VMSS uses `--zones 1 2 3`. All achievable, GCP is slightly cleaner.
>
> Mixed on-demand + Spot: AWS ASG has mixed instance policy with instance overrides. Azure VMSS Flexible has similar. GCP MIG supports Spot VMs alongside regular VMs. Feature parity.
>
> Autohealing: MIG and VMSS both have health-check-driven replacement. AWS ASG relies on EC2 or ELB health checks. All three converge on similar behavior.
>
> My honest take: for a greenfield project, MIG is marginally simpler to configure. For an inherited fleet, whichever the team already uses is fine — they're comparable in production."
