---
title: GCP Basics — Quick Orientation
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
order: 1
up: "[[GCP-MOC]]"
next: "[[GCP-01-Fundamentals]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# GCP Basics — Quick Orientation

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · → [[GCP-01-Fundamentals|GCP Fundamentals]]
<!-- /nav -->

Before diving into services, three things GCP does differently that shape everything downstream:

**1. Project is the atomic unit — not Account (AWS) or Subscription (Azure)**
Every GCP resource belongs to a Project. Billing, quotas, API enablement, IAM — all scoped to Project. Projects are cheap and free to create (unlike Azure Subscriptions which require billing setup). Multi-project architecture is the norm, not the exception.

**2. APIs must be explicitly enabled per project**
Unlike AWS/Azure where services are just available, GCP requires you to enable each API before use. `gcloud services enable compute.googleapis.com` before you can create a VM. Frustrating at first, security win in reality — reduces attack surface.

**3. Service Accounts are first-class identities**
In AWS you have Users and Roles. In Azure you have Users and Managed Identities. In GCP the default identity for anything programmatic is a Service Account. Every project gets a default compute SA automatically. This shapes IAM design fundamentally.

## Global Infrastructure

- **Regions**: `us-central1`, `europe-west1`, `asia-south1` (Mumbai), etc.
- **Zones**: `us-central1-a`, `us-central1-b`, `us-central1-c` (like AZs)
- **Multi-region locations**: `us`, `eu`, `asia` — used for Cloud Storage and BigQuery
- **Global services**: VPCs, Load Balancers, Cloud DNS — these span regions natively

## Product Naming Convention

GCP names services descriptively — "Cloud SQL", "Cloud Storage", "Cloud Functions", "Compute Engine". Once you internalize the pattern, service names tell you what they do. Compare to AWS's mix of literal (S3, EC2) and codenames (Kinesis, Athena, Redshift).
