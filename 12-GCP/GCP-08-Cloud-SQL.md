---
title: Cloud SQL
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
order: 9
up: "[[GCP-MOC]]"
prev: "[[GCP-07-MIG-Managed-Instance-Groups]]"
next: "[[GCP-09-Firestore-Bigtable-Spanner]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud SQL

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-07-MIG-Managed-Instance-Groups|MIG — Managed Instance Groups]] · → [[GCP-09-Firestore-Bigtable-Spanner|Firestore, Bigtable, Spanner]]
<!-- /nav -->

## Concept

Cloud SQL is GCP's managed relational database service for MySQL, PostgreSQL, and SQL Server.

## AWS Equivalent

**Cloud SQL = AWS RDS**

## Azure Equivalent

**Cloud SQL ≈ Azure Database for MySQL/PostgreSQL** (for those engines) OR **Azure SQL Database** (for SQL Server)

## Three-Way Comparison

| Feature | AWS RDS | Azure Database | Cloud SQL |
|---|---|---|---|
| Engines | MySQL, Postgres, MariaDB, SQL Server, Oracle | MySQL, Postgres, SQL Server (via Azure SQL) | MySQL, Postgres, SQL Server |
| HA model | Multi-AZ (sync standby) | Zone-redundant HA | Regional HA (sync replica in another zone) |
| Read replicas | Cross-region supported | Read replicas supported | Same-region or cross-region |
| Backup | Automated + manual snapshots | Automated + LTR | Automated + on-demand |
| Aurora equivalent | Aurora (custom) | Hyperscale/Business Critical | AlloyDB (Postgres-compatible, distinct product) |

## Key Differences from BOTH

**1. AlloyDB — the Postgres "Aurora Killer"**

GCP has a separate service: **AlloyDB for PostgreSQL**. It's a Postgres-compatible service with columnar accelerator, 4x faster than standard Postgres for transactional workloads, 100x faster for analytical queries, ~2x faster than AWS Aurora Postgres in Google's benchmarks.

Not covered under Cloud SQL — it's a distinct product. If you're doing Postgres at scale on GCP and RDS-style tuning isn't enough, AlloyDB is the answer. Similar architecture to Aurora (compute/storage separation).

**2. Regional Availability**

Cloud SQL HA uses a sync standby in a different zone within the same region. Like AWS RDS Multi-AZ. Failover is automatic and fast (~1 min).

**3. Public IP by Default (Change This)**

New Cloud SQL instances default to having a public IP. **Turn this off** and use Private IP + VPC Peering for internal access. Same principle as never exposing an RDS instance to the internet.

**4. Cloud SQL Auth Proxy**

Unique tool: `cloud-sql-proxy` runs locally or on a VM, authenticates to GCP via a Service Account, then presents a local socket that your app connects to. The proxy handles SSL and IAM auth transparently.

Great for: dev machines connecting to prod DBs securely, apps on Kubernetes that don't want to hardcode DB IPs, avoiding SSL certificate management.

AWS RDS Proxy is somewhat similar but positioned differently (mainly for connection pooling).

**5. IAM Authentication for Databases**

You can grant IAM users/service accounts direct database access — no separate DB password. `gcloud sql users create ... --type=cloud_iam_service_account` creates a DB user tied to a GCP identity.

Similar to AWS RDS IAM authentication and Azure Entra ID authentication for Azure SQL. GCP's setup is arguably simplest.

## Real Configuration

```bash
# Create a Postgres instance (development tier)
gcloud sql instances create prod-postgres \
  --project=myapp-prod \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-8192 \
  --region=us-central1 \
  --root-password='StrongPassword2026!' \
  --storage-size=100GB \
  --storage-type=SSD \
  --availability-type=REGIONAL \
  --no-assign-ip \
  --network=projects/myapp-prod/global/networks/prod-vpc

# Create a database
gcloud sql databases create appdb --instance=prod-postgres

# Create a user
gcloud sql users create appuser \
  --instance=prod-postgres \
  --password='UserPassword2026!'

# Create an IAM-based user (Service Account can auth directly)
gcloud sql users create web-sa@myapp-prod.iam \
  --instance=prod-postgres \
  --type=cloud_iam_service_account

# Configure automatic backups + point-in-time recovery
gcloud sql instances patch prod-postgres \
  --backup-start-time=03:00 \
  --enable-point-in-time-recovery \
  --retained-backups-count=30 \
  --retained-transaction-log-days=7

# Connect via Cloud SQL Auth Proxy (from any machine with gcloud auth)
cloud-sql-proxy --auto-iam-authn myapp-prod:us-central1:prod-postgres &
psql -h 127.0.0.1 -U web-sa@myapp-prod.iam -d appdb
```

## Production Use Case

**Production Postgres with HA + read replica for reporting:**

```
Project: myapp-prod-data
├── Cloud SQL: prod-postgres
│   ├── Version: Postgres 15
│   ├── Tier: db-custom-4-16384 (4 vCPU, 16GB RAM)
│   ├── Storage: 500GB SSD, auto-increase enabled
│   ├── HA: Regional (sync replica in us-central1-b)
│   ├── Backups: daily 03:00, 30-day retention, PITR enabled
│   ├── Network: Private IP only, VPC peering to prod-vpc
│   ├── Encryption: Google-managed keys (or CMEK for compliance)
│   └── IAM auth: web-sa@myapp-prod (least-privilege DB roles)
│
├── Read Replica: prod-postgres-reporting
│   ├── Region: us-central1 (same region, async replication)
│   └── Used by BI/analytics tools only
│
└── App tier connects via:
    ├── Cloud SQL Auth Proxy sidecar on Cloud Run
    ├── Or Private IP directly for VM-based apps
    └── Reporting queries use ApplicationName hint to route to replica
```

## Console Path

- Console → **SQL** → **Instances**
- Instance detail: Overview, Connections, Users, Databases, Backups, Replicas, Operations
- **Cloud SQL Studio** — in-browser SQL editor (recent addition, similar to Azure SQL Query Editor)

## Memory Hint

- **Cloud SQL** = "RDS but with the Auth Proxy for identity-based access"
- **AlloyDB** = Aurora Postgres equivalent (separate product, mention if asked)
- **Regional HA** = sync replica in another zone (like RDS Multi-AZ)
- **Auth Proxy** = unique GCP tool, worth knowing by name
- **Turn off public IP** on new instances immediately

## Interview Q&A — Cloud SQL

**Q: Design a highly available Postgres deployment on GCP. Compare to what you'd do on AWS and Azure.**

> "Cloud SQL Postgres with Regional (HA) availability. Setup:
>
> Instance tier: db-custom-4-16384 or higher based on load — Cloud SQL supports custom vCPU/RAM shapes like Compute Engine.
> Availability type: Regional — this provisions a synchronous standby in a different zone. Automatic failover in ~60 seconds if the primary zone fails.
> Storage: SSD with auto-increase enabled (up to configured max), 30-day PITR retention.
> Networking: Private IP only, exposed via VPC peering to the app VPC. No public endpoint.
> Auth: IAM database authentication with dedicated Service Accounts per app tier. No shared passwords.
> Backup: automated daily backups, PITR enabled for granular restore.
>
> Cross-cloud comparison:
>
> AWS RDS Postgres Multi-AZ is the direct analogue. Same architecture (sync standby, automatic failover), similar cost profile. RDS has been production-battle-tested longer, ecosystem is deeper. Cloud SQL is comparable in feature set for standard Postgres workloads.
>
> Azure Database for PostgreSQL Flexible Server with zone-redundant HA is the Azure equivalent. Feature parity roughly, cost model slightly different.
>
> Where GCP wins: if you need scale beyond what standard Postgres handles, AlloyDB for Postgres exists as a distinct service — think of it as GCP's Aurora Postgres. Compute-storage separation, columnar accelerator for OLAP, 2x faster than Aurora Postgres in benchmarks. If your workload is Postgres-heavy and needs Aurora-like scaling, GCP has a stronger story than raw RDS Postgres or Azure Flexible Server.
>
> For a straightforward Postgres deployment though, all three clouds deliver production-grade managed Postgres. Choose based on cloud-native integration (which cloud does the rest of your stack live in?), operational familiarity, and cost per your specific access patterns."
