---
title: S3 — Simple Storage Service
type: reference
module: aws
status: ✅ Active Reference
interview-relevance: 🔴 Maximum
tags:
  - aws
  - devops
  - cloud
  - interview
  - masterreview
order: 9
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-08-ASG]]"
next: "[[AWS-10-RDS]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# S3 — Simple Storage Service

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-08-ASG|ASG — Auto Scaling Group]] · → [[AWS-10-RDS|RDS — Relational Database Service]]
<!-- /nav -->

## Concept

**Amazon S3** is an object storage service — stores files (called "objects") in containers (called "buckets"). Highly durable, highly available, essentially unlimited storage.

**Not a filesystem** — no mount, no folders (though key names can mimic paths).

## Key Facts

```
Data model:       Object storage (not block, not file)
Bucket name:      Globally unique (no two accounts can have same name)
Region:           Buckets are region-specific
Object model:     Key-value pair (key = path, value = content)
Max object size:  5 TB single object
Multipart upload: Required for objects > 100MB, recommended > 5MB
Durability:       99.999999999% (11 nines) — practically indestructible
Availability:     99.99% (Standard class)
```

## Storage Types Comparison

```
Object Storage (S3)
    ├── Access via HTTP/HTTPS API (GET, PUT, DELETE)
    ├── No filesystem, no mount
    ├── Metadata + unique key + content
    └── Best for: unstructured data, backups, media, logs

Block Storage (EBS)
    ├── Raw disk blocks
    ├── Mounted as filesystem
    ├── Attached to one instance
    └── Best for: OS drives, databases

File Storage (EFS)
    ├── Shared filesystem
    ├── Multiple instances mount it simultaneously
    └── Best for: shared config, ML training data
```

## Common Use Cases

**Web Hosting:**

- Host static websites (HTML, CSS, JS)
- Combine with CloudFront for global CDN

**Data Lake:**

- Central repository for structured and unstructured data
- Used with Athena for SQL queries, Glue for ETL

**Backup and Disaster Recovery:**

- Store backups with 11 nines durability
- Combine with lifecycle policies for cost optimization
- Cross-region replication for DR

**Application Data:**

- User uploads (images, videos, documents)
- App logs (structured JSON logs archived to S3)
- ML training data

## S3 Versioning

**What it does:** Keeps every version of every object in the bucket.

```
Without versioning:
    Upload file.txt v1 → stored
    Upload file.txt v2 → v1 GONE, replaced

With versioning:
    Upload file.txt v1 → stored with version ID
    Upload file.txt v2 → BOTH versions exist
    Delete file.txt    → delete marker added, file not truly gone
                         Restore by removing delete marker
```

**Why enable:**

- Accidental deletion recovery
- Rollback to previous version
- Ransomware protection
- Audit trail

**Cost consideration:** Versioning multiplies storage. Use lifecycle policies to expire old versions.

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# List versions of an object
aws s3api list-object-versions --bucket my-bucket --prefix file.txt

# Get a specific version
aws s3api get-object --bucket my-bucket --key file.txt --version-id abc123 file.txt
```

## S3 Replication

**Two types:**

```
Same-Region Replication (SRR)
    ├── Copy objects to another bucket in same region
    ├── Use for: compliance, data segregation, log aggregation
    └── Example: replicate production logs to dedicated analytics account

Cross-Region Replication (CRR)
    ├── Copy objects to bucket in different region
    ├── Use for: disaster recovery, geo-redundancy, latency reduction
    └── Example: primary bucket in ap-south-1, replicated to us-east-1
```

**Requirements:**

- Versioning must be enabled on BOTH buckets
- Objects uploaded AFTER enabling replication are copied (existing objects need manual sync)
- IAM role for S3 to perform replication

## Data Encryption

**Two levels:**

**Encryption at Rest:**

```
SSE-S3 (AWS-managed keys)
    ├── S3 manages encryption keys
    ├── Simplest, most common
    └── No extra cost

SSE-KMS (KMS-managed keys)
    ├── Uses AWS KMS
    ├── More control, audit trail
    ├── Cost: KMS API calls
    └── Best for: compliance requirements

SSE-C (Customer-provided keys)
    ├── You provide the key on each request
    ├── AWS doesn't store the key
    └── Rare — most teams use SSE-KMS

Client-Side Encryption
    ├── Encrypt before uploading to S3
    ├── AWS never sees plaintext
    └── Highest security, most complex
```

**Encryption in Transit:**

- Enforce HTTPS via bucket policy
- SSL/TLS for all uploads and downloads

**Best practice:** Enable SSE-KMS by default on all buckets.

## S3 Bucket Policies

**JSON-based access control attached directly to a bucket.**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website-bucket/*"
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-website-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

**Common actions:**

- `s3:GetObject` — download/read
- `s3:PutObject` — upload/write
- `s3:DeleteObject` — delete
- `s3:ListBucket` — list contents

## S3 Storage Classes

```
Standard
    ├── Frequently accessed data
    ├── Millisecond retrieval
    ├── 99.99% availability
    └── Highest cost per GB

Intelligent-Tiering
    ├── Auto-moves objects between tiers based on access
    ├── Best for unknown access patterns
    ├── Small monitoring fee per object
    └── No retrieval fees

Standard-IA (Infrequent Access)
    ├── Less frequently accessed but rapid retrieval when needed
    ├── Lower storage cost, retrieval fee
    ├── 99.9% availability
    └── Minimum 30-day storage

One Zone-IA
    ├── Same as IA but single AZ (less resilient)
    ├── 20% cheaper than Standard-IA
    └── OK if you can regenerate the data

Glacier Instant Retrieval
    ├── Archives with millisecond retrieval
    ├── Lower storage cost than IA
    └── Minimum 90-day storage

Glacier Flexible Retrieval (formerly Glacier)
    ├── Retrieval: minutes to hours
    ├── Very low storage cost
    └── Minimum 90-day storage

Glacier Deep Archive
    ├── Retrieval: 12+ hours
    ├── Cheapest — < $1 per TB per month
    └── Minimum 180-day storage
```

**Decision matrix:**

```
Accessed daily      → Standard
Accessed weekly     → Intelligent-Tiering
Accessed monthly    → Standard-IA
Accessed yearly     → Glacier Instant Retrieval
Accessed rarely     → Glacier Flexible
Almost never        → Glacier Deep Archive
```

## S3 Lifecycle Policies

**Automate object management** — move to cheaper storage as data ages, delete when not needed.

**Transition Actions:**

```
Day 0     → Standard (upload)
Day 30    → Standard-IA (transition)
Day 90    → Glacier Flexible (transition)
Day 365   → Glacier Deep Archive (transition)
```

**Expiration Actions:**

```
Day 365       → Delete objects
Day 30 (version) → Delete old versions after 30 days
```

**Example policy:**

```json
{
  "Rules": [{
    "Id": "ArchiveOldLogs",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"}
    ],
    "Expiration": {"Days": 365}
  }]
}
```

**Real production impact:** Storing 100TB of logs

- All in Standard: $2,300/month
- With lifecycle to Glacier after 90 days: $200/month
- Same data, 92% cost reduction

## Hosting Static Website

**Full workflow:**

```
1. Create S3 bucket (name = domain e.g. www.mysite.com)
2. Upload website files (index.html, styles.css, script.js)
3. Enable Static Website Hosting
   ├── Properties → Static Website Hosting → Enable
   ├── Index document: index.html
   └── Error document: error.html
4. Disable "Block Public Access"
5. Add bucket policy for public read access
6. (Optional) Attach CloudFront for CDN + HTTPS
7. (Optional) Route 53 alias record to point domain to S3
```

**Real world:** Most static sites use S3 + CloudFront + Route 53. S3 alone is HTTP-only — you need CloudFront for HTTPS.

## S3 Snow Family

**Physical devices for moving huge amounts of data to AWS when internet upload is impractical.**

```
AWS Snowcone (small)
    ├── 8 TB storage
    ├── Portable, ruggedized
    └── Use: field data collection, edge computing

AWS Snowball Edge (medium)
    ├── 80 TB storage
    ├── Also does edge compute
    └── Use: moving petabytes to AWS

AWS Snowmobile (massive — truck!)
    ├── Up to 100 PB per truck
    ├── Literal shipping container on a truck
    └── Use: moving entire data centers to cloud
```

**When to use:** If uploading via internet would take weeks or months, order a Snow device. AWS ships it, you fill it, AWS ships it back and uploads to S3.

## S3 Storage Gateway

**Hybrid cloud storage:** Connects on-premises environments to S3 in the cloud.

**Three types:**

```
File Gateway
    ├── NFS/SMB mount from on-prem servers
    ├── Files stored in S3 behind the scenes
    └── Use: seamless cloud backup for existing file servers

Volume Gateway
    ├── iSCSI block storage from on-prem
    ├── Backed by S3, snapshots to EBS
    └── Use: cloud-backed block storage

Tape Gateway
    ├── Virtual tape library, backed by S3 and Glacier
    └── Use: replacing physical tape backup systems
```

## Practical Commands

```bash
# List all buckets
aws s3 ls

# List contents of a bucket
aws s3 ls s3://my-bucket/

# Recursive list
aws s3 ls s3://my-bucket/ --recursive

# Upload file
aws s3 cp file.txt s3://my-bucket/

# Download file
aws s3 cp s3://my-bucket/file.txt ./

# Sync local dir to S3 (like rsync)
aws s3 sync ./dist s3://my-bucket/

# Sync S3 to local
aws s3 sync s3://my-bucket/ ./backup/

# Delete file
aws s3 rm s3://my-bucket/file.txt

# Delete bucket and all contents (dangerous!)
aws s3 rb s3://my-bucket --force

# Set bucket-wide encryption
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

## Real Production Setup

**Terraform state bucket:**

```
Bucket: mycompany-terraform-state
    ├── Versioning: ENABLED (rollback state)
    ├── Encryption: SSE-KMS
    ├── Bucket policy: only specific IAM roles allowed
    ├── Access logging: enabled
    └── Lifecycle: expire old versions after 90 days
```

**Application logs:**

```
Bucket: mycompany-app-logs
    ├── App writes JSON logs to S3 every hour
    ├── Prefix: /year/month/day/hour/
    ├── Standard class initially
    ├── Lifecycle: Standard-IA after 30 days, Glacier after 90 days
    └── Delete after 2 years
```

## Memory Hints

- **S3** = infinite HDD in the sky
- **Bucket** = folder at the top level (globally named)
- **Object** = a file (with key + metadata + content)
- **Versioning** = undo button for the whole bucket
- **Lifecycle** = automatic file aging (move to cold storage as it gets old)
- **Storage Classes** = different pricing tiers based on access pattern

## Interview Q&A — S3

**Q: How do you decide which S3 storage class to use?**

> "Two factors: access frequency and cost tolerance for retrieval.
>
> For frequently accessed data — anything hit multiple times per month per object — use Standard. It's the most expensive per GB but no retrieval fees. Application uploads, active user data, current logs.
>
> For infrequently accessed data — hit once a month or less — use Standard-IA. About 45% cheaper storage than Standard, but you pay a small retrieval fee per GB downloaded. Good for logs older than 30 days that you occasionally need.
>
> For archives — data you might need but rarely — use Glacier variants. Glacier Instant Retrieval keeps millisecond access but at cheaper storage. Glacier Flexible has minutes-to-hours retrieval and is much cheaper. Glacier Deep Archive is the cheapest at under $1 per TB per month, but retrieval takes 12+ hours.
>
> Best practice: don't manually decide. Use lifecycle policies. Objects start in Standard, transition to Standard-IA after 30 days, Glacier after 90, Deep Archive after 365. Delete after retention period. Automated cost optimization.
>
> One more option: Intelligent-Tiering. AWS monitors access patterns and moves objects automatically. Costs a small monitoring fee per object but eliminates guesswork. Best when access patterns are unpredictable."

**Q: Explain S3 versioning and when you'd enable it.**

> "Versioning keeps every version of every object. Without versioning, uploading a file with the same key overwrites the previous version — the old one is gone. With versioning, both versions exist, each with a unique version ID.
>
> Even deletion doesn't truly delete — S3 adds a delete marker. The object versions still exist, and you can restore by removing the delete marker.
>
> When to enable: First, on Terraform state buckets — absolutely critical. Corrupted state file can be rolled back to previous version. Never lose infrastructure state. Second, on application configuration buckets — accidental config push can be reverted instantly. Third, on any bucket with critical data where accidental deletion would be catastrophic.
>
> Cost consideration: every version consumes storage. A 1GB file overwritten 100 times = 100GB stored. Combine versioning with lifecycle policy: expire old versions after 30-90 days depending on retention needs.
>
> Ransomware protection is a modern reason. If attacker encrypts your S3 objects, previous versions are still there. MFA Delete adds another layer — even you can't permanently delete versions without MFA."

**Q: How do you host a static website on S3? What are the limitations?**

> "The workflow: create bucket, upload HTML/CSS/JS files, enable static website hosting in bucket properties, disable Block Public Access, apply a bucket policy that grants s3:GetObject to everyone. The bucket exposes a URL like `mybucket.s3-website.ap-south-1.amazonaws.com`.
>
> Limitations of S3 alone: HTTP only — no HTTPS support directly from S3 website endpoint. This is a dealbreaker for most modern sites. No custom SSL certificates. Regional endpoint — users far from the bucket's region see slower load times. No advanced features like caching headers control, WAF integration, or edge computing.
>
> Production pattern: S3 + CloudFront + Route 53. S3 hosts the files with public access disabled (only CloudFront can access). CloudFront serves the content globally via CDN with HTTPS. ACM provides free SSL certificate. Route 53 provides the domain and routes users to CloudFront.
>
> This gives you: HTTPS, global CDN, custom domain, DDoS protection, edge caching, WAF integration. Same static S3 files, production-ready delivery. Cost is minimal — CloudFront's free tier covers a lot, and S3 storage for a static site is often < $1/month."

**Q: How do you protect an S3 bucket from public exposure?**

> "Multiple layers of defense — never rely on just one.
>
> Layer 1: Block Public Access at the account level. Since 2023, AWS blocks public access by default for new buckets. This is the safety net that catches accidental public settings.
>
> Layer 2: Bucket policies. Don't include statements with `Principal: '*'` unless intentional for a public website. Use IAM users, roles, or specific AWS accounts as principals.
>
> Layer 3: ACLs. Legacy access control — most teams disable ACLs entirely now with 'Bucket owner enforced' setting.
>
> Layer 4: Encryption. Enable default SSE-KMS encryption. Even if data is exposed, it's encrypted at rest.
>
> Layer 5: Access logging. Enable S3 access logs to another bucket. Every request is logged — you can audit who accessed what.
>
> Layer 6: AWS Config rules. Set up automated compliance checks that flag any bucket that becomes public or lacks encryption.
>
> Real world scanning: AWS Trusted Advisor and third-party tools like Prowler scan for misconfigurations. Every quarter, run a full audit — buckets with public access, encryption disabled, cross-account access. Fix findings immediately.
>
> Most-cited S3 data leaks in the news were misconfigured bucket policies. Always assume attackers are scanning for open buckets — they are."
