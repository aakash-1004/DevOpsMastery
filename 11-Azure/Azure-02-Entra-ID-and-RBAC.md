---
title: "Entra ID & RBAC"
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
order: 2
up: "[[Azure-MOC]]"
prev: "[[Azure-01-Fundamentals]]"
next: "[[Azure-03-Virtual-Machines]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Entra ID & RBAC

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-01-Fundamentals|Azure Fundamentals]] · → [[Azure-03-Virtual-Machines|Virtual Machines]]
<!-- /nav -->

## Concept

Azure splits identity and authorization into two systems:
- **Microsoft Entra ID** (formerly Azure Active Directory / Azure AD) — identity provider
- **Azure RBAC** — authorization on Azure resources

Entra ID handles WHO. RBAC handles WHAT they can do.

## AWS Equivalent

| AWS | Azure |
|---|---|
| IAM Users | Entra ID Users |
| IAM Groups | Entra ID Groups |
| IAM Roles (assumable) | Entra ID Service Principals + Managed Identities |
| IAM Policies (JSON) | RBAC Role Assignments |
| AWS SSO | Entra ID (natively an IdP) |

**The big shift:** AWS IAM is one system doing both identity and authorization. Azure splits it. This matters because Entra ID also handles Microsoft 365, third-party SaaS SSO, and enterprise-wide identity — it's much bigger than just cloud resource access.

## Key Differences from AWS

**1. Role-Based, not Policy-Based**
AWS IAM: you write JSON policies with `Action` and `Resource`.
Azure: you assign **pre-defined roles** (Contributor, Reader, Owner) to a **scope** (subscription / RG / resource).

You *can* write custom roles in JSON, but 90% of production uses built-in roles. This is simpler than AWS IAM once you learn the vocabulary.

**2. Scope is Hierarchical and Inherited**
```
Assignment at Subscription scope  → applies to all RGs and resources
Assignment at RG scope            → applies to all resources in that RG
Assignment at Resource scope      → applies to just that one resource
```
In AWS you'd write a policy with a `Resource` ARN pattern. In Azure you pick where to attach the role assignment.

**3. Managed Identities Beat IAM Roles**
Azure's answer to "IAM role attached to EC2" is **Managed Identity**. Two flavors:
- **System-assigned**: tied to one resource's lifetime — VM deleted, identity gone
- **User-assigned**: standalone identity, attach to multiple resources

Same purpose as AWS Instance Profiles but cleaner API — no STS token juggling in your app code, the SDK just works.

**4. Conditional Access = MFA on Steroids**
Entra ID has policies like "require MFA when accessing from outside corporate network" or "block sign-in from unmanaged devices." Much richer than AWS IAM's MFA setting.

## Built-in RBAC Roles You Must Know

| Role | Scope of Power | AWS Analogue |
|---|---|---|
| **Owner** | Full access + can grant access to others | Admin |
| **Contributor** | Full access, CANNOT grant access | PowerUser |
| **Reader** | View everything, change nothing | ReadOnlyAccess |
| **User Access Administrator** | Only manages access, nothing else | Split of IAM admin |

Then hundreds of service-specific roles: `Virtual Machine Contributor`, `Storage Blob Data Reader`, `AKS Cluster Admin Role`, etc.

## Real Configuration

```bash
# List all users
az ad user list --output table

# Create a user
az ad user create \
  --display-name "Aakash Rao" \
  --user-principal-name aakash@yourdomain.onmicrosoft.com \
  --password 'TempPass123!' \
  --force-change-password-next-sign-in true

# Assign Contributor role at RG scope
az role assignment create \
  --assignee aakash@yourdomain.onmicrosoft.com \
  --role "Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/rg-dev-sandbox

# Assign at subscription scope (broader)
az role assignment create \
  --assignee aakash@yourdomain.onmicrosoft.com \
  --role "Reader" \
  --scope /subscriptions/{sub-id}

# List role assignments for a user
az role assignment list --assignee aakash@yourdomain.onmicrosoft.com --output table

# Create a Service Principal (like AWS IAM Role for CI/CD)
az ad sp create-for-rbac \
  --name "github-actions-deployer" \
  --role Contributor \
  --scopes /subscriptions/{sub-id}/resourceGroups/rg-prod-web

# This returns appId, password, tenant — save these; they're your credentials
```

## Production Use Case

**Scenario:** Your team of 8 needs Azure access. Two developers need to deploy to dev, two SREs need prod access, one manager needs read-only across everything.

**Wrong way (AWS habit):** create 8 individual role assignments per RG per person = 40+ assignments.

**Right way:**
1. Create Entra ID **groups**: `grp-developers`, `grp-sres`, `grp-managers`
2. Add users to groups
3. Assign roles to groups at appropriate scope:
   - `grp-developers` → Contributor on `rg-dev-*` RGs
   - `grp-sres` → Contributor on `rg-prod-*` RGs
   - `grp-managers` → Reader at subscription scope
4. Onboarding a new engineer = add them to a group. Done.

**Bonus production pattern:** Use **Privileged Identity Management (PIM)** — engineers get elevated permissions only when needed, with approval workflow, audit trail, and auto-expiry. No standing prod admin access. Not free (requires Entra ID P2 license) but critical for compliance environments.

## Console Path

- Search bar → "Microsoft Entra ID" → **Users** (create/manage identities)
- Any resource / RG / subscription → **Access control (IAM)** blade → **Role assignments** tab
- The "IAM" blade name is confusing — Microsoft uses "IAM" for Azure RBAC UI even though the identity itself lives in Entra ID

## Memory Hint

- **Entra ID** = who you are (identity)
- **RBAC** = what you can do (authorization)
- **Role** = a bundle of permissions (pre-defined, don't write JSON unless you must)
- **Scope** = where the role applies (sub / RG / resource)
- **Managed Identity** = "instance profile for Azure" — no credentials to manage
- "AWS says: write a policy. Azure says: pick a role and stick it somewhere."

## Interview Q&A — Entra ID & RBAC

**Q: An AWS engineer joins your Azure-heavy team. Walk them through the mental shift for identity and access.**

> "Three shifts to internalize.
>
> First, identity and authorization are separate systems in Azure. Entra ID owns identity — users, groups, service principals — and it's a full enterprise identity provider used well beyond cloud resource access. RBAC handles authorization on Azure resources. In AWS both are IAM.
>
> Second, Azure is role-first, not policy-first. You rarely write JSON. You assign pre-built roles like Contributor, Reader, Storage Blob Data Contributor to a scope. The scope is hierarchical: subscription > resource group > resource, and assignments cascade down. This is simpler than AWS IAM in practice — less permission-crafting, more picking from a catalog.
>
> Third, Managed Identities replace IAM roles for compute. When you turn on a system-assigned managed identity on a VM, Azure creates a service principal tied to that VM's lifetime. Your app calls Azure SDK, the SDK grabs a token from the instance metadata endpoint automatically. No secrets in code, no STS assume-role dance. Cleaner than AWS instance profiles in my opinion.
>
> One gotcha for AWS folks: Azure has TWO 'admin' concepts — Owner and Contributor. Contributor has full resource access but CAN'T grant access to others. Owner can. This split doesn't exist in AWS's AdministratorAccess. Use Contributor by default; reserve Owner for people who actually need to manage permissions."

**Q: Design a least-privilege setup for a GitHub Actions pipeline deploying to Azure.**

> "Create a Service Principal specifically for the pipeline — not a human user. That gives you a clean audit trail and no shared credentials.
>
> `az ad sp create-for-rbac` scoped to just the resource groups the pipeline touches. Not subscription-wide. If it deploys to `rg-prod-web-eastus`, that's the only scope it gets.
>
> Role choice depends on what the pipeline does. If it's Terraform managing existing resources, Contributor on those RGs. If it also creates role assignments or Key Vault access policies, upgrade to User Access Administrator on top — or better, split into a two-stage pipeline where the identity-changing stage runs with a separately-scoped SP.
>
> Store the SP credentials in GitHub Actions secrets. Modern practice: use OpenID Connect federation instead of long-lived secrets — GitHub Actions gets a short-lived token from Entra ID per run. Same pattern as AWS OIDC federation to IAM roles. No password in secrets, less to rotate.
>
> For extra safety: enable Entra ID sign-in logs and set up alerts on unusual SP activity — logins from unexpected IPs, permission escalation attempts. Same defense-in-depth mindset as AWS CloudTrail."

**Q: When would you write a custom RBAC role instead of using a built-in one?**

> "Almost never — but here's when.
>
> Built-in roles cover 95% of cases. The catalog has hundreds — 'Storage Blob Data Reader,' 'AKS Cluster User Role,' 'DNS Zone Contributor.' Search before you write custom.
>
> Custom roles make sense when the built-in role is too broad but the next-narrower one is too narrow. Real example: a backup automation needs to create VM snapshots and read blob storage, but nothing else. 'Contributor' is too broad — it can also delete VMs. There's no built-in that combines just those two capabilities. So you write a custom role with exactly `Microsoft.Compute/snapshots/write` + `Microsoft.Storage/storageAccounts/blobServices/read`.
>
> Downsides of custom roles: harder to audit ('who has this role and what does it grant?'), version drift across environments, and they don't automatically get new sub-permissions when Azure adds new APIs. So the operational tax is real.
>
> Interview trap: candidates who default to custom roles look inexperienced. The right instinct is 'built-in first, custom only when justified.'"
