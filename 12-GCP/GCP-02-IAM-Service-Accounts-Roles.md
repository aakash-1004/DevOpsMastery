---
title: "IAM — Identity, Service Accounts, Roles"
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
order: 3
up: "[[GCP-MOC]]"
prev: "[[GCP-01-Fundamentals]]"
next: "[[GCP-03-Compute-Engine]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# IAM — Identity, Service Accounts, Roles

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-01-Fundamentals|GCP Fundamentals]] · → [[GCP-03-Compute-Engine|Compute Engine]]
<!-- /nav -->

## Concept

Cloud IAM (Identity and Access Management) is GCP's authorization system. It determines who (identity) can do what (role) on which resource.

## AWS Equivalent

**GCP IAM ≈ AWS IAM** (but with different structure — no policies-as-JSON in the same way)

## Azure Equivalent

**GCP IAM ≈ Azure RBAC** (structurally more similar to Azure than AWS)

Like Azure, GCP is role-based, not policy-based. You bind roles to identities at a scope. No hand-writing JSON policies unless you're doing custom roles.

## Three-Way Comparison

| Concept | AWS IAM | Azure RBAC | GCP IAM |
|---|---|---|---|
| Identity types | User, Role | User, Service Principal, Managed Identity | User, Service Account, Group |
| Permission bundle | Policy (JSON) | Role (predefined + custom) | Role (predefined + custom) |
| Attachment model | Policy → User/Role | Role Assignment → Identity + Scope | Role Binding → Identity + Resource |
| Scope hierarchy | Per-resource ARN | Sub → RG → Resource | Org → Folder → Project → Resource |
| Compute identity | Instance Profile (IAM Role) | Managed Identity | Service Account |

## Key Differences from AWS AND Azure

**1. Everything Programmatic Uses Service Accounts**

In AWS, an EC2 instance uses an Instance Profile (a wrapper around IAM Role). In Azure, a VM uses a Managed Identity. In GCP, a VM uses a **Service Account** — but Service Accounts are their own top-level identity type, not derived from user accounts.

Every project gets a **default Compute Engine service account** automatically. VMs, functions, GKE pods — they all authenticate as some Service Account. This makes SAs first-class citizens in a way IAM Roles and Managed Identities aren't quite.

**2. Three Tiers of Roles**

| Tier | Purpose | Example |
|---|---|---|
| **Basic roles** | Very broad, project-wide | `roles/owner`, `roles/editor`, `roles/viewer` |
| **Predefined roles** | Service-scoped, granular | `roles/compute.admin`, `roles/storage.objectViewer` |
| **Custom roles** | You define exact permissions | `roles/myorg.customRole` |

**Never use Basic roles in production.** They're too broad. `roles/editor` is essentially "admin on everything." Interview red flag if a candidate defaults to basic roles.

**3. Bindings Instead of Assignments**

GCP calls the "grant this role to this identity on this resource" operation a **binding**, not an assignment (Azure) or an attachment (AWS). Semantically identical to Azure — attach a role to an identity at a scope. The scope is any resource in the hierarchy (Org, Folder, Project, or specific resource).

**4. Service Account Impersonation**

A distinctive GCP pattern: users can **impersonate** Service Accounts. Instead of downloading a service account key file (bad practice — long-lived credential), you grant your user the `roles/iam.serviceAccountTokenCreator` role on the SA, then run:

```bash
gcloud auth application-default login --impersonate-service-account=my-sa@project.iam.gserviceaccount.com
```

Your commands execute as the SA without you holding its key. Same idea as AWS `sts:AssumeRole` but cleaner day-to-day.

**5. IAM Conditions**

GCP supports conditional role bindings using CEL (Common Expression Language) — grant access only when specific conditions are met (time of day, IP range, resource tag). Similar to AWS IAM policy conditions but structured differently.

## Real Configuration

```bash
# List current identity
gcloud auth list

# Create a service account
gcloud iam service-accounts create ci-deployer \
  --display-name="CI Deployment SA" \
  --description="Used by GitHub Actions to deploy"

# Grant SA a role at project scope
gcloud projects add-iam-policy-binding myapp-prod \
  --member="serviceAccount:ci-deployer@myapp-prod.iam.gserviceaccount.com" \
  --role="roles/run.admin"

# Grant a user impersonation rights on the SA (preferred over key files)
gcloud iam service-accounts add-iam-policy-binding \
  ci-deployer@myapp-prod.iam.gserviceaccount.com \
  --member="user:aakash@aakashrao.dev" \
  --role="roles/iam.serviceAccountTokenCreator"

# Impersonate the SA for CLI commands
gcloud auth application-default login --impersonate-service-account=ci-deployer@myapp-prod.iam.gserviceaccount.com

# Grant a group at folder scope (cascades to all projects under folder)
gcloud resource-manager folders add-iam-policy-binding 123456789 \
  --member="group:sre-team@aakashrao.dev" \
  --role="roles/compute.viewer"

# Create a custom role (rare — usually predefined is enough)
gcloud iam roles create backupOperator \
  --project=myapp-prod \
  --title="Backup Operator" \
  --description="Can trigger backups and read logs" \
  --permissions="compute.snapshots.create,logging.logEntries.list" \
  --stage=GA
```

## Production Use Case

**Team of 10 needs multi-project access, CI/CD deploys, audit trail:**

```
Organization: aakashrao.dev
├── Folder: engineering
│   ├── Project: myapp-prod
│   ├── Project: myapp-staging
│   └── Project: myapp-dev
│
├── Google Group: developers@aakashrao.dev
│   └── Binding: roles/editor on projects myapp-dev + myapp-staging directly
│
├── Google Group: sres@aakashrao.dev
│   └── Binding: roles/owner on myapp-prod (with break-glass audit alerting)
│   └── Binding: roles/viewer on myapp-dev, myapp-staging
│
├── Service Account: ci-deployer@myapp-prod.iam.gserviceaccount.com
│   ├── Binding: roles/run.admin, roles/cloudsql.client on myapp-prod
│   └── Federated identity to GitHub Actions via Workload Identity Federation
│       (no key files, no long-lived credentials)
│
└── Break-glass: 2 named users with roles/owner directly on Organization
    (audited quarterly, alerts on any use)
```

Same principles as AWS multi-account IAM or Azure RBAC — use groups not individuals, scope narrowly, prefer short-lived credentials.

## Console Path

- Console → **IAM & Admin** (from hamburger menu)
  - **IAM** — all bindings on the current project
  - **Service Accounts** — create/manage SAs
  - **Roles** — see all roles, create custom
  - **Audit Logs** — every IAM change is logged
  - **Workload Identity Federation** — federate external identities (GitHub, AWS, Azure) to GCP SAs

## Memory Hint

- **GCP IAM** = "Azure RBAC but with Service Accounts everywhere"
- **Basic roles** = never use (too broad)
- **Predefined roles** = default choice (`roles/compute.admin`, `roles/storage.objectViewer`)
- **Service Account** = the identity for anything programmatic
- **Impersonation** = the modern pattern (no key files)
- "Every project has a default compute SA. Every VM authenticates as some SA. GCP is Service-Account-first."

## Interview Q&A — IAM

**Q: Compare AWS IAM, Azure RBAC, and GCP IAM. Which model does GCP most resemble?**

> "GCP IAM is structurally closer to Azure RBAC than to AWS IAM. Both use role-based, not policy-based, models. You bind pre-built roles to identities at a scope. Custom roles exist but are the exception.
>
> The key differences from Azure:
>
> First, the scope hierarchy is different. Azure: Subscription → Resource Group → Resource. GCP: Organization → Folder → Project → Resource. The extra layers give more granularity for large orgs.
>
> Second, Service Accounts are more first-class in GCP. Azure has Managed Identities, which are lifecycle-tied to specific resources. GCP Service Accounts are standalone identities that any resource can use — a VM, a GKE pod, a Cloud Function. You create them explicitly and grant them roles just like you would a user.
>
> Third, three tiers of roles: basic, predefined, custom. Basic (owner/editor/viewer) is too broad for production. Predefined roles are the sweet spot — `roles/compute.instanceAdmin` for VM management, `roles/storage.objectViewer` for read access to buckets. Custom is rare.
>
> Compared to AWS IAM: GCP is much less about writing JSON policies. AWS forces you to think in terms of Actions and Resources at the JSON level. GCP mostly abstracts that away — you pick roles from a catalog and bind them.
>
> Practical migration mindset: an AWS engineer moving to GCP should stop reaching for JSON immediately. Search the predefined role catalog first. 90% of what you need already exists."

**Q: How would you securely wire up a GitHub Actions pipeline to deploy to GCP?**

> "Workload Identity Federation, no service account keys. This is the modern pattern and it's genuinely elegant.
>
> Traditional approach (bad): create a service account, download its key file as JSON, store in GitHub Secrets. Problems: long-lived credential, key rotation is manual, leaked key is a nightmare, key file works from anywhere.
>
> Modern approach: federate GitHub's OIDC tokens to a GCP Service Account.
>
> Setup:
> 1. Create a Workload Identity Pool in GCP
> 2. Add a GitHub OIDC provider to the pool: configures GCP to trust tokens issued by GitHub for a specific repo
> 3. Create a GCP Service Account for the deploy pipeline
> 4. Grant the Service Account the specific IAM roles it needs on the target project
> 5. Bind the SA to allow impersonation from GitHub Actions running in the specific repo
>
> Runtime: GitHub Action requests an OIDC token from GitHub, GCP validates it via the federation, exchanges it for a short-lived Service Account access token. Every workflow run uses a fresh 1-hour token. No secrets in GitHub except a public identifier.
>
> AWS equivalent: OIDC federation from GitHub to an IAM Role. Azure equivalent: OIDC federation from GitHub to a Service Principal via Entra ID. Same pattern in all three clouds — GCP arguably has the smoothest developer experience for setting it up."
