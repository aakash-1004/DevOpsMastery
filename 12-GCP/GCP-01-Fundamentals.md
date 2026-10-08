---
title: GCP Fundamentals
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
order: 2
up: "[[GCP-MOC]]"
prev: "[[GCP-00-Quick-Orientation]]"
next: "[[GCP-02-IAM-Service-Accounts-Roles]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# GCP Fundamentals

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-00-Quick-Orientation|GCP Basics — Quick Orientation]] · → [[GCP-02-IAM-Service-Accounts-Roles|IAM — Identity, Service Accounts, Roles]]
<!-- /nav -->

Before touching any service, understand GCP's resource hierarchy. This is the single biggest mental-model shift — and unlike Azure where Resource Groups are the surprise concept, in GCP it's how deeply **Project** sits at the center of everything.

## The Hierarchy

```
Organization                    (root — one per company, tied to Google Workspace/Cloud Identity)
    ↓
Folder                          (optional grouping — like Azure Management Groups)
    ↓
Project                         (billing + quota + API + IAM boundary — the workhorse)
    ↓
Resource                        (VM, bucket, DB — everything lives in a project)
```

## Key Concepts

**Organization**
- One per company, tied to your domain (`aakashrao.dev`)
- Free to create if you have Google Workspace or Cloud Identity
- The IAM root — organization-level policies cascade down
- Analogous to AWS Organizations root OR Azure Entra ID Tenant

**Folder**
- Optional intermediate layer for grouping projects
- Common patterns: folder per department (`engineering`, `finance`) or per environment (`prod`, `dev`, `sandbox`)
- IAM policies at folder level inherit to all child projects
- Analogous to Azure Management Groups (AWS has OUs)

**Project**
- **The load-bearing concept in GCP.** More like Azure Subscription than AWS Account in terms of power, but with a critical difference: **projects are cheap and instantly created**
- Billing account attached (multiple projects can share one billing account)
- Quotas are per-project (VMs, IPs, API calls)
- APIs enabled per-project (`gcloud services enable ...`)
- IAM policies attach to project (or higher — inherited down)
- Every resource has a project ID as part of its identity
- Project ID must be globally unique across all of GCP (like an S3 bucket name)

**Resource**
- The actual thing — VM, bucket, database
- Belongs to exactly one project
- Full resource name: `projects/{project-id}/zones/{zone}/instances/{vm-name}`

## Three-Way Hierarchy Comparison

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Company root | Organization | Entra ID Tenant | Organization |
| Grouping layer | Organizational Units (OUs) | Management Groups | Folders |
| Billing boundary | Account | Subscription | Project |
| Logical container | (tags + IAM) | Resource Group | Project (again — it's both) |
| Resource | Resource | Resource | Resource |

**The key GCP quirk:** Project plays TWO roles that are split in Azure. It's both the billing boundary (like Subscription) AND the logical grouping container (like Resource Group). GCP doesn't have a separate "resource group" concept — you create more projects instead.

## Why "Just Create More Projects" Is the GCP Way

In AWS, you tag resources and use IAM for grouping.
In Azure, you create Resource Groups within Subscriptions.
In GCP, you create separate projects: `myapp-prod`, `myapp-dev`, `myapp-sandbox`.

Because projects are:
- Free to create
- Delete cleanly (nuking a project nukes all resources — the atomic-cleanup superpower)
- Isolated by default (no cross-project access without explicit IAM grants)

...multi-project architectures are the default pattern, not a governance overhead.

## Real Configuration

```bash
# Login
gcloud auth login

# List existing projects
gcloud projects list

# Create a new project
gcloud projects create myapp-dev-aakash \
  --name="MyApp Dev" \
  --folder=123456789012

# Set default project for gcloud
gcloud config set project myapp-dev-aakash

# Link billing account to project (required before creating billable resources)
gcloud beta billing projects link myapp-dev-aakash \
  --billing-account=01ABCD-234567-EFGH89

# Enable APIs (each service you'll use needs its API enabled)
gcloud services enable \
  compute.googleapis.com \
  storage.googleapis.com \
  container.googleapis.com \
  sqladmin.googleapis.com

# List enabled APIs
gcloud services list --enabled

# Delete a project (30-day grace period, then permanent deletion)
gcloud projects delete myapp-dev-aakash
```

## Regions and Zones

- Regions: `us-central1` (Iowa), `us-east1` (South Carolina), `europe-west1` (Belgium), `asia-south1` (Mumbai), `asia-southeast1` (Singapore)
- Zones: `us-central1-a`, `us-central1-b`, `us-central1-c` — always at least 3 zones per region
- Location for storage/BigQuery: `us` (multi-region), `us-central1` (single region), `nam4` (dual-region North America)

Different from Azure: **all GCP regions have zones**. No exceptions where a region only has one location. Simpler HA design.

## Console Path

- Portal: `console.cloud.google.com`
- Top navbar: **Project selector** (this is where you switch projects — always check what project you're in)
- Left navigation: hamburger menu → categorized services (Compute, Storage, Databases, Networking, etc.)
- **Cloud Shell** (top right icon) — free Debian VM with gcloud/kubectl/terraform pre-installed, browser-based terminal
- **APIs & Services** → enable APIs, view usage
- **Billing** → cost breakdown by project/service/label

## Memory Hint

- **Organization** = the company root (like AWS Org, Azure Tenant)
- **Folder** = optional grouping (like OU or Management Group)
- **Project** = billing + quota + API + IAM — GCP's "load-bearing" concept
- **Projects are cheap** — the GCP idiom is "create more projects," not "use one big project"

"AWS uses one account with tags. Azure uses one subscription with resource groups. GCP just creates a new project."

## Interview Q&A — Fundamentals

**Q: You've worked with both AWS and Azure. Walk me through the equivalent hierarchy in GCP and where it's genuinely different.**

> "GCP has Organization → Folder → Project → Resource. Organization is the company root, tied to your Google Workspace domain. Folders are optional grouping — think of them like OUs in AWS Organizations or Management Groups in Azure. Both AWS and Azure engineers should recognize this three-way similarity so far.
>
> The interesting difference is Project. In GCP, Project plays two roles that Azure splits between Subscription and Resource Group. It's the billing boundary AND the logical container. GCP doesn't have a separate 'resource group' concept — the answer is 'just create more projects.'
>
> Why this matters: projects are free to create and instantly provisioned. You don't provision a new Azure Subscription for every environment — that requires billing setup, admin approval, and paperwork. But in GCP, `gcloud projects create myapp-dev` and you have a fresh, isolated environment in 10 seconds.
>
> Practical implication: multi-project architectures are the norm. A typical setup might have `myapp-prod`, `myapp-staging`, `myapp-dev`, `myapp-sandbox`, plus shared projects like `myapp-shared-network` and `myapp-shared-security`. Each project cleanly isolated with its own IAM policies. Deleting a project nukes everything cleanly.
>
> Interview trap: candidates from AWS backgrounds sometimes try to run everything in one project because 'that's what we did in AWS.' Senior signal is knowing GCP encourages splitting into more projects for isolation."

**Q: Explain the mental shift from 'AWS account' to 'GCP project' for a team migrating.**

> "AWS accounts are heavyweight — creating one involves AWS Organizations, root user setup, billing consolidation, Control Tower guardrails if you're doing it right. Teams keep the number small and use IAM + tags within each account.
>
> GCP projects are lightweight. You can have hundreds. A team migrating from AWS should NOT try to map one AWS account to one GCP project — that misses the model entirely.
>
> The right mapping: one AWS account often becomes multiple GCP projects. If in AWS you separated prod and dev via naming conventions and IAM within a single account, in GCP you'd separate them into two projects with independent IAM policies. If in AWS you used AWS Organizations with a prod account and dev account, that's a closer 1:1 to GCP projects.
>
> Practical migration pattern: start by categorizing AWS resources by lifecycle and blast radius. Anything sharing the same lifecycle (staging web tier + staging DB tier + staging cache) can live in one GCP project. Different environments = different projects.
>
> Cost implication: since projects are free, over-splitting has no financial cost. Under-splitting has isolation cost — a bad IAM binding can affect more resources. Err on the side of more projects.
>
> Bonus: use Folders to group related projects and apply org-wide policies. `engineering` folder holds all engineering projects; a folder-level IAM policy grants the engineering team default access to everything under it."
