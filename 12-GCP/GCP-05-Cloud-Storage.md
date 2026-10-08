---
title: Cloud Storage
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
order: 6
up: "[[GCP-MOC]]"
prev: "[[GCP-04-Persistent-Disk]]"
next: "[[GCP-06-Cloud-Load-Balancing]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud Storage

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-04-Persistent-Disk|Persistent Disk]] · → [[GCP-06-Cloud-Load-Balancing|Cloud Load Balancing]]
<!-- /nav -->

## Concept

Cloud Storage is GCP's object storage — store files as objects, access via HTTP API, essentially unlimited scale.

## AWS Equivalent

**Cloud Storage = S3**

## Azure Equivalent

**Cloud Storage = Azure Blob Storage** (but simpler — no Storage Account wrapper)

## Three-Way Comparison

| Concept | AWS S3 | Azure Blob | GCP Cloud Storage |
|---|---|---|---|
| Container name | Bucket | Container (in Storage Account) | Bucket |
| Namespace | Global | Per-storage-account | **Global** |
| Wrapper resource | None | Storage Account | None |
| Location model | Region | Region | Region OR multi-region OR dual-region |
| Signed URLs | Pre-signed URL | SAS token | Signed URL |
| Storage classes | 6 (Standard, IA, One Zone-IA, Glacier IR, Glacier FR, Glacier DA) | 4 (Hot, Cool, Cold, Archive) | 4 (Standard, Nearline, Coldline, Archive) |

## Key Differences from BOTH

**1. Buckets Are NOT Wrapped in an Account**

Azure requires a Storage Account, then containers inside it. GCP is like S3 — buckets are top-level objects in a project. Simpler.

Bucket names are globally unique across all of GCP (like S3 bucket names). Reserve early.

**2. Location Types: Region / Dual-Region / Multi-Region**

Unique GCP flexibility:
- **Region** (`us-central1`) — cheapest, single-region durability
- **Dual-region** (`nam4` = us-central1 + us-east1) — synced across two regions, moderate cost, good DR
- **Multi-region** (`us`, `eu`, `asia`) — spread across all regions in the geography, highest durability, most expensive

AWS S3 has only region + Cross-Region Replication (async). Azure Blob has ZRS/GRS/GZRS as tiers within a storage account. GCP's location model is a distinct axis independent of storage class.

**3. Storage Classes**

| GCP | AWS | Azure | Retrieval time |
|---|---|---|---|
| Standard | S3 Standard | Hot | Immediate |
| Nearline | S3 Standard-IA | Cool | Immediate (30-day min) |
| Coldline | S3 Glacier Instant Retrieval | Cold | Immediate (90-day min) |
| Archive | S3 Glacier Deep Archive | Archive | Hours (365-day min) |

All GCP classes support instant retrieval except Archive. AWS Glacier tiers historically had multi-hour retrieval; GCP has always emphasized instant-access even for cold tiers.

**4. Object Lifecycle Rules**

Same concept as S3 lifecycle policies. Transition objects between classes based on age, delete after N days, etc.

**5. Object Versioning + Retention Policies**

Same as S3 versioning. Additionally, GCP supports **Bucket Lock** — an immutable retention policy where objects cannot be deleted before a retention period expires, even by admins. Good for compliance (WORM — Write Once Read Many).

## Real Configuration

```bash
# Create a bucket (name globally unique, choose location and class)
gsutil mb -p myapp-prod -l us-central1 -c standard gs://myapp-prod-user-uploads

# Or with gcloud
gcloud storage buckets create gs://myapp-prod-user-uploads \
  --project=myapp-prod \
  --location=us-central1 \
  --default-storage-class=standard \
  --uniform-bucket-level-access

# Upload a file
gcloud storage cp local-file.txt gs://myapp-prod-user-uploads/

# Or via gsutil (older CLI, still widely used)
gsutil cp local-file.txt gs://myapp-prod-user-uploads/

# List objects
gcloud storage ls gs://myapp-prod-user-uploads/

# Download
gcloud storage cp gs://myapp-prod-user-uploads/local-file.txt .

# Generate a signed URL (5 min TTL)
gcloud storage sign-url gs://myapp-prod-user-uploads/local-file.txt \
  --duration=5m \
  --impersonate-service-account=web-sa@myapp-prod.iam.gserviceaccount.com

# Set lifecycle policy via JSON
cat > lifecycle.json <<EOF
{
  "lifecycle": {
    "rule": [
      { "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30} },
      { "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90} },
      { "action": {"type": "Delete"},
        "condition": {"age": 2555} }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://myapp-prod-user-uploads/

# Enable versioning
gsutil versioning set on gs://myapp-prod-user-uploads/

# Set retention policy (compliance WORM)
gsutil retention set 7y gs://myapp-prod-user-uploads/
gsutil retention lock gs://myapp-prod-user-uploads/   # Cannot be undone
```

## Uniform vs Fine-Grained Access Control

Buckets have two ACL modes:
- **Uniform (recommended)** — all objects inherit bucket-level IAM. No per-object ACLs. Simpler, cleaner. Default for new buckets.
- **Fine-grained** — legacy per-object ACLs on top of IAM. More flexible but complex; hard to audit.

Always create new buckets with `--uniform-bucket-level-access`. AWS/Azure engineers should reach for this to match the "bucket policy only" mental model.

## Production Use Case

**User upload pipeline with least-privilege access:**

```
Project: myapp-prod-data
├── Bucket: myapp-prod-user-uploads (us-central1, Standard, uniform ACL)
│   ├── Lifecycle: to Nearline after 30 days, Coldline after 90, delete after 7 years
│   ├── Versioning: on
│   ├── Encryption: CMEK via KMS key in same region
│   └── Access: web-sa@myapp-prod (roles/storage.objectAdmin scoped to this bucket)
│
├── Bucket: myapp-prod-public-assets (multi-region us, Standard)
│   ├── Public read via allUsers with roles/storage.objectViewer
│   └── Cloud CDN in front for caching
│
├── Bucket: myapp-prod-backups (dual-region nam4)
│   ├── Bucket Lock: 7-year retention (immutable)
│   └── Access: backup-sa only, no user access
│
Web tier (Cloud Run):
├── Uses SA web-sa
├── Upload flow: backend generates signed URL, client uploads to bucket directly
└── Download flow: backend generates signed URL for read, client downloads directly
```

Same architecture as an S3-based flow. No account keys, all signed URLs, immutable audit trail.

## Console Path

- Console → **Cloud Storage** → **Buckets**
- Bucket detail:
  - **Objects** — browse and upload
  - **Configuration** — location, storage class, versioning
  - **Permissions** — IAM bindings
  - **Protection** — versioning, retention, lifecycle
  - **Lifecycle** — rules for transitions and deletions

## Memory Hint

- **Cloud Storage** = "S3 without the annoying region-name-in-URL, with more location flexibility"
- **Bucket names global** — reserve like S3
- **Multi-region location** = maximum durability, HIGHER cost
- **Dual-region** = the sweet spot for regional DR
- **Uniform access** = default for new buckets, use it

## Interview Q&A — Cloud Storage

**Q: A team wants to migrate from S3 to Cloud Storage. What are the operational surprises?**

> "Migration itself is straightforward — both are HTTP object stores with similar APIs. `gsutil` even has an S3-compatible mode. The interesting operational surprises are elsewhere:
>
> First, location model. S3 is per-region. Cloud Storage adds dual-region and multi-region options that don't exist in S3 (S3 replication is separate and async). For workloads that need cross-region durability, GCP simplifies this — pick a dual-region location and you get sync replication built in. AWS engineers usually configure CRR separately.
>
> Second, storage classes look similar but have different minimum storage durations. AWS S3 Intelligent-Tiering automatically transitions based on access patterns. GCP has Autoclass which is similar but requires opt-in per bucket. Similar outcomes, different opt-in models.
>
> Third, retrieval behavior for cold tiers is fundamentally different. AWS S3 Glacier historically had multi-hour retrieval — retrieving an archived object was an async operation. GCP Coldline and Archive support instant retrieval (Archive charges more for it, but no wait). This changes application design — you can query archived data in Cloud Storage without a two-step retrieval pattern.
>
> Fourth, uniform access control. GCP encourages bucket-level IAM only, no per-object ACLs. AWS S3 has bucket policies AND ACLs, and teams often end up with both which creates auditing nightmares. GCP's uniform mode forces cleaner design.
>
> Migration mechanics: `gsutil rsync` from S3 to Cloud Storage handles most workloads. Storage Transfer Service (a GCP-managed service) handles large migrations with parallelism and resumability. For petabyte-scale, use Transfer Appliance for physical shipment."

**Q: Design a WORM-compliant archive on Cloud Storage.**

> "Bucket Lock with retention policy is the answer. This is GCP's WORM (Write Once Read Many) primitive.
>
> Setup:
> 1. Create a bucket, dual-region or multi-region for durability
> 2. Storage class Archive (cheapest for long-term retention)
> 3. Enable versioning (so overwrites don't destroy data)
> 4. Set retention policy: `gsutil retention set 7y gs://audit-archive/` (7 years, adjust for regulatory requirement)
> 5. LOCK the retention policy: `gsutil retention lock gs://audit-archive/` — this is irreversible, no admin can shorten it after locking
> 6. Enable object holds for specific critical objects that need even more protection
>
> Result: objects cannot be deleted or modified before 7 years pass. Even project owners cannot bypass. This meets SEC 17a-4, FINRA, and most WORM compliance standards.
>
> Comparison:
> - AWS S3 Object Lock with Compliance mode is the direct equivalent
> - Azure Blob Immutable storage with Legal Hold is the Azure equivalent
> - All three achieve similar compliance. GCP's Bucket Lock is arguably the simplest to configure.
>
> Cost optimization: Archive class + multi-region location + versioning. Total cost per TB per month is a few dollars — cheap for the compliance value.
>
> Watch out for: locked retention policies cannot be undone. Test in a non-production bucket first. And separate the auditors' write access from the readers' read access — no single identity should be able to both write and read compliance data."
