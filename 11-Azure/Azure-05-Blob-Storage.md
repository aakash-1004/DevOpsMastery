---
title: Blob Storage
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
order: 5
up: "[[Azure-MOC]]"
prev: "[[Azure-04-Managed-Disks]]"
next: "[[Azure-06-Load-Balancer-App-Gateway-Front-Door]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Blob Storage

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-04-Managed-Disks|Managed Disks]] · → [[Azure-06-Load-Balancer-App-Gateway-Front-Door|Load Balancer, App Gateway, Front Door]]
<!-- /nav -->

## Concept

Azure Blob Storage is object storage — same category as AWS S3 or GCP Cloud Storage. Store files as objects, access via HTTP API, essentially unlimited scale.

## AWS Equivalent

**Azure Blob Storage = AWS S3**

Same category, similar features, different terminology and naming rules.

## Key Differences from AWS

**1. Storage Account Wraps Everything**
This is a real mental-model shift. AWS: you create a bucket. Azure: you create a **Storage Account** first, and inside that account you create **containers** (which are like buckets).

```
Storage Account (globally unique name, has performance tier, redundancy, endpoints)
├── Blob Service
│   ├── Container 1  (like an S3 bucket — but scoped to this account)
│   ├── Container 2
│   └── Container 3
├── File Service (SMB shares — like AWS EFS)
├── Queue Service (like AWS SQS)
└── Table Service (basic NoSQL key-value — cheap alternative to Cosmos DB)
```

A single Storage Account can host all four service types. This is why the naming levels are: `storage_account.blob.core.windows.net/container/blob.png`.

**2. Three Blob Types**
- **Block blobs** — regular files (99% of use cases). Equivalent to S3 objects.
- **Append blobs** — optimized for append-only logs
- **Page blobs** — random-access, used for VHDs (Azure Managed Disks are actually page blobs under the hood)

For interviews and DevOps day-to-day: block blobs. Others exist, know they exist, move on.

**3. Access Tiers**

| Azure Tier | AWS S3 Class | Min Storage Duration |
|---|---|---|
| **Hot** | Standard | None |
| **Cool** | Standard-IA | 30 days |
| **Cold** | Glacier Instant Retrieval | 90 days |
| **Archive** | Glacier Deep Archive | 180 days |

Same lifecycle logic as S3: hot data cheap to access, cold data cheap to store expensive to access. **Archive** requires rehydration (up to 15 hours) before you can read the blob — same as Glacier Deep Archive.

**4. Redundancy Options (Not Just One Region)**

| Option | Copies | Location | AWS Equivalent |
|---|---|---|---|
| **LRS** | 3 | Single AZ | No direct AWS parallel |
| **ZRS** | 3 | Three AZs | S3 (default multi-AZ) |
| **GRS** | 6 | Primary region + paired region | S3 Cross-Region Replication |
| **RA-GRS** | 6 | Same as GRS + read from secondary | S3 CRR with cross-region reads |
| **GZRS** | 6 | ZRS + geo-redundant | Best combined durability |

You pick this at Storage Account level. Changing later requires migration effort. Default in new accounts: `RA-GRS` or `GRS` depending on region.

**5. SAS Tokens Are the Common Access Pattern**
Shared Access Signature (SAS) tokens are Azure's answer to S3 pre-signed URLs — time-limited, scope-limited URL that grants access without needing IAM credentials.

Three types:
- **User Delegation SAS** — signed by Entra ID credentials (preferred, auditable)
- **Service SAS** — signed by account key (works but less secure)
- **Account SAS** — broadest scope, use sparingly

## Real Configuration

```bash
# Create storage account (name must be 3-24 chars, lowercase, unique globally)
az storage account create \
  --name stdevopsaakash01 \
  --resource-group rg-storage-lab \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

# Get account key
az storage account keys list --resource-group rg-storage-lab --account-name stdevopsaakash01

# Create a container
az storage container create \
  --name uploads \
  --account-name stdevopsaakash01 \
  --auth-mode login

# Upload a file
az storage blob upload \
  --account-name stdevopsaakash01 \
  --container-name uploads \
  --name hello.txt \
  --file ./hello.txt \
  --auth-mode login

# List blobs
az storage blob list \
  --account-name stdevopsaakash01 \
  --container-name uploads \
  --auth-mode login \
  --output table

# Download
az storage blob download \
  --account-name stdevopsaakash01 \
  --container-name uploads \
  --name hello.txt \
  --file ./downloaded.txt \
  --auth-mode login

# Generate a SAS token for a specific blob (read-only, 1 hour)
az storage blob generate-sas \
  --account-name stdevopsaakash01 \
  --container-name uploads \
  --name hello.txt \
  --permissions r \
  --expiry $(date -u -d '1 hour' '+%Y-%m-%dT%H:%MZ') \
  --auth-mode login \
  --as-user
```

## Production Use Case

**Application file uploads with least-privilege access:**

```
Storage Account: stprodwebfiles (GZRS for durability)
├── Container: user-uploads (private)
├── Container: processed (private)
└── Container: public-assets (public read via CDN)

App tier (AKS):
├── Managed Identity assigned
├── Role: Storage Blob Data Contributor scoped to user-uploads
└── Uploads directly via SDK, no keys

Client browsers:
├── Backend generates User Delegation SAS token (5 min TTL)
└── Client uploads directly to blob via SAS URL

Lifecycle:
├── uploads/*: Hot for 30 days → Cool for 90 → Archive after 1 year
└── Delete after 7 years (compliance retention)
```

Note: **no account keys anywhere**. Backend uses managed identity, clients get short-lived SAS tokens signed by user delegation. Same maturity level as AWS S3 with IAM roles + presigned URLs.

## Console Path

- Portal → **Storage accounts** (search)
- Storage account detail:
  - **Containers** (left blade) → click into container → upload/manage blobs
  - **Access keys** — the account keys (avoid using directly)
  - **Shared access signature** — generate SAS
  - **Configuration** — access tier, redundancy
  - **Lifecycle management** — rules for tier transitions and expiration
  - **Data protection** — versioning, soft delete, immutability

## Memory Hint

- **Storage Account** = the AWS "bucket" concept split into two layers (account + container)
- **Hot / Cool / Cold / Archive** — temperature = access frequency (hot = frequent)
- **LRS < ZRS < GRS < GZRS** — increasing redundancy, increasing cost
- **SAS = pre-signed URL** — time-limited, scoped, safer than sharing keys

## Interview Q&A — Blob Storage

**Q: Compare Blob Storage to S3. What surprises AWS engineers?**

> "Three things.
>
> First, the Storage Account layer. In AWS you just create a bucket. In Azure you first create a Storage Account with performance tier and redundancy settings, THEN create containers inside it. The Storage Account is the pricing and redundancy boundary; containers are just organization within it. AWS engineers try to create 'buckets' and get confused by the wrapper.
>
> Second, redundancy is more granular. AWS S3 is multi-AZ by default in a region, with CRR as an add-on for cross-region. Azure gives you five choices — LRS, ZRS, GRS, RA-GRS, GZRS — spanning single-AZ up to multi-region-multi-AZ. You explicitly pick this per storage account. It's more control but more decisions.
>
> Third, SAS tokens are richer than S3 pre-signed URLs. You can generate SAS at container level (not just blob), grant multiple permissions in one token, and use User Delegation SAS which is signed by Entra ID credentials so it's auditable per user. AWS pre-signed URLs are simpler but less granular.
>
> Practical gotcha: Storage Account names must be globally unique across all of Azure — like S3 bucket names. Reserve yours early or you'll fight for names. Standard practice: prefix with `st` (for storage), then a domain identifier."

**Q: Design a static website hosting solution on Azure — HTTPS, custom domain, global CDN. Compare to AWS S3+CloudFront."**

> "The Azure equivalent of S3 + CloudFront + Route 53 + ACM is Blob Storage + Front Door + Azure DNS + managed certificates.
>
> Storage Account with 'Static website' feature enabled — this creates a `$web` container automatically and gives you a primary endpoint URL. Upload your build artifacts there.
>
> Azure Front Door in front of the storage account. Front Door is Azure's global L7 load balancer / CDN combined — think CloudFront + AWS Global Accelerator merged into one product. Origin points to the static website endpoint. Front Door handles global caching at 100+ edge POPs, HTTPS with managed certs (free), and WAF.
>
> Azure DNS zone for the domain. CNAME `www` and Alias record on the apex pointing to Front Door.
>
> Managed certificate on Front Door for HTTPS — auto-renewing, no ACM equivalent to manage.
>
> Result: HTTPS static site with global CDN, custom domain, DDoS protection via Front Door, sub-100ms load times worldwide. Same as the AWS pattern. Slight edge to Azure here — Front Door bundles CDN + WAF + global load balancing in one service, whereas in AWS you compose CloudFront + WAF + Route 53 separately.
>
> Cost: for a small static site, both are under $10/month. Front Door pricing is a bit different — you pay per rule and per request, plus data egress. CloudFront's free tier is more generous for hobby projects."

**Q: How do you protect a Blob container from accidental exposure?**

> "Layered defense, same principle as AWS S3 protection.
>
> Layer 1: Storage account level — disable public blob access at the account. Under Configuration, set 'Allow Blob public access' to Disabled. This overrides container-level settings — nothing in this account can be public. Analogous to S3 'Block Public Access' at account level.
>
> Layer 2: Network restrictions. Configure Storage Firewall to allow access only from specific VNets and IP ranges. For internal apps, use Private Endpoints — the storage account becomes reachable only via a private IP inside your VNet, not the public endpoint at all.
>
> Layer 3: Identity-based access only. Turn off Shared Key access if possible. Force everything to use Entra ID authentication. This makes SAS tokens and managed identities the only access paths, and every access is auditable to a specific identity.
>
> Layer 4: Encryption. Enabled by default at rest with Microsoft-managed keys. For compliance workloads, use customer-managed keys (CMK) via Azure Key Vault — same pattern as S3 with KMS.
>
> Layer 5: Monitoring. Enable diagnostic logs to Log Analytics. Alert on unusual patterns: mass downloads, access from unexpected countries, sudden spike in failed auth attempts. Same defense-in-depth as AWS S3 with CloudTrail.
>
> Real practice: I audit storage accounts monthly with Azure Policy — enforce 'no public blob access' as a compliance rule, and any non-compliant account gets flagged automatically. Same as AWS Config rules for S3 public access."
