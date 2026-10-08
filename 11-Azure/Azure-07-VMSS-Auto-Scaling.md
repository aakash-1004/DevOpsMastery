---
title: VMSS — Auto Scaling
type: reference
module: azure
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - azure
  - devops
  - cloud
  - multicloud
  - interview
  - masterreview
order: 7
up: "[[Azure-MOC]]"
prev: "[[Azure-06-Load-Balancer-App-Gateway-Front-Door]]"
next: "[[Azure-08-Azure-SQL-Database]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# VMSS — Auto Scaling

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-06-Load-Balancer-App-Gateway-Front-Door|Load Balancer, App Gateway, Front Door]] · → [[Azure-08-Azure-SQL-Database|Azure SQL Database]]
<!-- /nav -->

## Concept

Virtual Machine Scale Sets — deploy and manage a fleet of identical VMs that scale in/out based on demand.

## AWS Equivalent

**VMSS = AWS Auto Scaling Group + Launch Template combined**

Same fundamental purpose. Slightly different mental model — VMSS is a single resource that encompasses both the "how to launch" definition and the "how many to run" logic. AWS separates Launch Templates from ASGs; Azure unifies them.

## Key Differences from AWS

**1. VMSS Modes: Uniform vs Flexible**

- **Uniform orchestration** — every VM identical, single VM size, faster scale, less flexibility. Older model.
- **Flexible orchestration** — mix VM sizes, spot + on-demand together, more like modern ASG mixed-instance policy. **Prefer this for new deployments.**

**2. Native Azure Integration**
VMSS integrates directly with:
- Load Balancer / App Gateway backends (auto-registers)
- Managed Identity (whole scale set gets one identity)
- Application Health extension (custom health signals)
- Automatic OS image updates (rolling patching built-in)

**3. Autoscale Rules Are More Expressive**
Azure autoscale supports multiple simultaneous rules with time-based schedules and can consume Application Insights custom metrics natively. Similar power to AWS but different syntax.

## Real Configuration

```bash
# Create VMSS with autoscaling
az vmss create \
  --resource-group rg-prod-web-eastus \
  --name vmss-web-prod \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-prod-eastus \
  --subnet snet-web \
  --load-balancer lb-web \
  --upgrade-policy-mode automatic \
  --orchestration-mode Flexible \
  --zones 1 2 3

# Add autoscale profile
az monitor autoscale create \
  --resource-group rg-prod-web-eastus \
  --resource vmss-web-prod \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-web \
  --min-count 3 \
  --max-count 20 \
  --count 3

# Add scale-out rule: CPU > 70% for 5 min → add 2 instances
az monitor autoscale rule create \
  --resource-group rg-prod-web-eastus \
  --autoscale-name autoscale-web \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Add scale-in rule: CPU < 30% for 10 min → remove 1 instance
az monitor autoscale rule create \
  --resource-group rg-prod-web-eastus \
  --autoscale-name autoscale-web \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1
```

## Production Use Case

**Web tier with predictable business-hours pattern:**

```
VMSS: vmss-prod-web (Flexible orchestration, 3 AZs)
├── Baseline: 5 instances (24/7)
├── Business hours (9AM-6PM IST weekdays): scale to 15 minimum
├── Reactive scaling:
│   ├── CPU > 70% for 5 min → +3 instances (aggressive scale-out)
│   ├── HTTP request rate > 500/sec per instance → +5 instances
│   └── CPU < 30% for 15 min → -1 instance (gentle scale-in)
├── Max cap: 40 instances (cost guardrail)
└── Health check: App Gateway probe on /health, 15s interval

Rolling upgrade policy: 20% batch, automatic if health probe passes
Application Health extension reports app-level health → VMSS replaces bad instances
```

## Console Path

- Portal → **Virtual machine scale sets**
- Overview → Instance count, running/stopped counts, load balancer status
- **Scaling** blade → autoscale rules, schedules
- **Instances** blade → list of individual VMs, can SSH, reimage, upgrade individually
- **Upgrade policy** blade → automatic / manual / rolling settings

## Memory Hint

- **VMSS** = "fleet of clones with autoscale" (AWS ASG merged with Launch Template)
- **Flexible** = use this (mixed sizes, spot, modern)
- **Uniform** = don't use (legacy, single size only)
- **Cool-down** ≈ Azure's version of AWS scaling cooldown, prevents thrashing

## Interview Q&A — VMSS

**Q: Coming from AWS Auto Scaling Groups, what's the mental model shift for VMSS?**

> "AWS separates concerns: Launch Template defines what an instance looks like, ASG defines scaling policy. Two resources referencing each other.
>
> VMSS bundles both. One resource contains the VM spec (image, size, extensions, network config) and the scaling behavior. Simpler to think about in some ways, less composable in others.
>
> Practical differences that matter day-to-day:
>
> First, orchestration mode. Choose Flexible for new deployments — supports mixed VM sizes and spot instances in the same set, closer to modern ASG mixed-instance policy. Uniform is single-size only, older, still around for compatibility.
>
> Second, rolling upgrades are first-class. You update the VMSS model (change image, extension config, size) and it rolls the change through instances in configurable batches. AWS you'd script this or use CodeDeploy. Azure builds it in.
>
> Third, health signals. Beyond load balancer health probes, VMSS has the Application Health extension — your app reports health to a URL, the extension reports to VMSS which uses it for instance replacement decisions. More granular than the classic 'ELB health check' pattern.
>
> Fourth, autoscale rules can reference custom Application Insights metrics natively — scale based on queue depth, active user sessions, business metrics. AWS supports this via CloudWatch custom metrics, but the wiring is cleaner in Azure."

**Q: You inherit a VMSS running Uniform orchestration in production. Would you migrate it to Flexible? What's the process?**

> "Yes, I'd migrate it — but carefully. Flexible unlocks spot integration, mixed VM sizes, and better zone distribution. Long-term maintenance cost of staying on Uniform is higher.
>
> Migration path: you can't in-place convert. You deploy a new Flexible VMSS in parallel, weight traffic between them via the load balancer, then decommission the old one.
>
> Steps:
> 1. Provision the new Flexible VMSS with matching configuration
> 2. Add it to the same load balancer backend pool alongside the existing Uniform set
> 3. Verify traffic hitting both, both pass health checks
> 4. Gradually reduce Uniform capacity while increasing Flexible capacity
> 5. Once Flexible carries all traffic and has been stable for a week, remove Uniform from backend, then delete
>
> Watch out for: any external dependencies referencing specific instance names or IDs. VMSS instance names change per set. Anything hard-coded needs updating.
>
> If the environment has strict downtime requirements, plan the migration during a low-traffic window and have rollback ready — the old VMSS stays intact until you're sure the new one is stable."
