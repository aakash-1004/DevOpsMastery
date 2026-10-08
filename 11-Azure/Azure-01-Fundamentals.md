---
title: Azure Fundamentals
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
order: 1
up: "[[Azure-MOC]]"
next: "[[Azure-02-Entra-ID-and-RBAC]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure Fundamentals

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · → [[Azure-02-Entra-ID-and-RBAC|Entra ID & RBAC]]
<!-- /nav -->

Before any service, you need to understand Azure's resource hierarchy. This is **the single biggest mental-model shift from AWS** and gets asked in interviews constantly.

## The Hierarchy

```
Microsoft Entra ID Tenant  (identity boundary — like an AWS Organization)
    ↓
Management Group           (group of subscriptions with shared policy)
    ↓
Subscription               (billing + quota boundary — like an AWS Account)
    ↓
Resource Group             (logical container — NO AWS equivalent)
    ↓
Resource                   (VM, storage account, DB, etc.)
```

## Key Concepts

**Tenant (Entra ID)**
- The identity root — one company = one tenant, usually
- Contains users, groups, applications
- Not billing-related; purely identity

**Subscription**
- The **billing container** — every resource belongs to a subscription
- Also a quota boundary (per-subscription limits on VMs, IPs, etc.)
- Closest analogue: an AWS Account
- Enterprises typically use multiple: `prod-sub`, `dev-sub`, `sandbox-sub`

**Resource Group (RG)**
- **This concept doesn't exist in AWS.** Critical to understand.
- A logical folder holding related resources (VM + its disk + NIC + public IP all live together)
- Resources CAN'T span RGs, but CAN span regions within one RG
- Deleting the RG deletes all resources in it — **incredibly useful for cleanup**
- Every resource must belong to exactly one RG

**Resource**
- The actual thing: VM, storage account, database, VNet, etc.
- Every resource has a globally unique Resource ID: `/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm-name}`

## AWS ↔ Azure Hierarchy Mapping

| AWS | Azure |
|---|---|
| Organization | Entra ID Tenant + Management Groups |
| Account | Subscription |
| (implicit — you use tags) | **Resource Group** |
| Resource (EC2, S3 bucket) | Resource (VM, Storage Account) |
| Region | Region (mostly same concept) |
| Availability Zone | Availability Zone (same concept, but not all regions support them) |

## Why Resource Groups Matter (a lot)

In AWS, you use tags + IAM to organize resources logically. In Azure, the Resource Group **is** the organizational primitive.

Real production patterns:
```
rg-prod-web-eastus         → all production web tier resources in East US
rg-prod-db-eastus          → all production database resources in East US
rg-dev-sandbox-aakash      → developer sandbox, deletable anytime
```

**Killer feature:** `az group delete --name rg-dev-sandbox-aakash` nukes everything inside — 47 resources gone in one command. In AWS you'd script this or hit "delete everything" one resource type at a time.

## Regions and Availability Zones

- Regions: `eastus`, `westeurope`, `centralindia`, `southeastasia`, etc.
- AZs: numbered 1, 2, 3 within a region (like AWS)
- **Not all Azure regions have AZs** — check before designing for HA
- Some regions have paired regions built in for DR (e.g., East US ↔ West US)

## Real Configuration

```bash
# Login (interactive)
az login

# List subscriptions
az account list --output table

# Set active subscription
az account set --subscription "MSDN-Subscription-Name"

# Create a resource group
az group create --name rg-devops-learning --location eastus

# List resource groups
az group list --output table

# Delete a resource group (and everything in it — dangerous!)
az group delete --name rg-devops-learning --yes --no-wait

# List all resources in an RG
az resource list --resource-group rg-devops-learning --output table
```

## Console Path

**Azure Portal home** — `portal.azure.com`
Everything anchors from here. Key spots to know:
- **All resources** (left sidebar) — flat list of everything
- **Resource groups** — grouped view, matches how you actually think
- **Subscriptions** (search bar) — billing and quota view
- **Microsoft Entra ID** (search bar) — identity

## Memory Hint

- **Tenant** = the whole company (identity)
- **Subscription** = the credit card (billing + quota)
- **Resource Group** = the project folder (organization)
- **Resource** = the actual thing

"AWS has accounts + tags. Azure has subscriptions + resource groups + tags."
The RG replaces half of what tags do in AWS.

## Interview Q&A — Fundamentals

**Q: You've worked in AWS. Walk me through the equivalent hierarchy in Azure and where they differ.**

> "AWS is Organization → Account → Resources, with tags providing logical grouping. Azure inserts an extra layer: Tenant → Subscription → Resource Group → Resource.
>
> The Tenant is Entra ID — pure identity boundary, one per company usually. Subscription is closest to an AWS Account — it's the billing container and quota boundary. Enterprises use multiple subscriptions to separate prod, dev, sandbox — same pattern as AWS multi-account strategies.
>
> The genuinely new concept is Resource Group. It's a mandatory logical container — every resource must live in exactly one RG. In AWS you fake this with tags plus IAM boundaries. In Azure it's structural.
>
> Practical implications: RGs enable atomic cleanup — `az group delete` removes 50 resources in one command. They also naturally scope Role-Based Access Control — grant Contributor role at RG level and it inherits down. In AWS you'd write more granular IAM policies to achieve the same.
>
> The trap for AWS engineers moving to Azure: over-nesting resources into too few RGs. Best practice: one RG per environment per tier — `rg-prod-web-eastus`, `rg-prod-db-eastus`. Not one giant `rg-production` with everything mixed."

**Q: Can a resource span multiple resource groups or regions?**

> "A resource lives in exactly one RG and one region. Non-negotiable at creation time. You can move resources between RGs later — Azure supports it for most resource types with `az resource move` — but at any moment a resource has one parent RG.
>
> Regions are more flexible. Different resources in one RG can be in different regions. So `rg-prod-webapp` might have VMs in East US and a Storage Account in Central India. This differs from AWS where region choice is per-resource and you don't have the RG concept forcing you to think about it.
>
> Practical note: keeping related resources in the same region matters more than keeping them in the same RG. Cross-region latency and egress costs will kill you fast if you mix regions carelessly."
