---
title: Azure SQL Database
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
order: 8
up: "[[Azure-MOC]]"
prev: "[[Azure-07-VMSS-Auto-Scaling]]"
next: "[[Azure-09-Cosmos-DB]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Azure SQL Database

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-07-VMSS-Auto-Scaling|VMSS — Auto Scaling]] · → [[Azure-09-Cosmos-DB|Cosmos DB]]
<!-- /nav -->

## Concept

Azure SQL Database is the managed relational database service — Microsoft's flagship, deeply tuned. Runs SQL Server engine as a service.

## AWS Equivalent

**Azure SQL Database ≈ AWS RDS for SQL Server, but more service-integrated**

For MySQL/PostgreSQL, Azure has separate services: **Azure Database for MySQL** and **Azure Database for PostgreSQL**. They're managed like RDS but the flagship (Azure SQL) is what interviews mostly focus on.

## Key Differences from AWS

**1. Three Deployment Options**

| Option | AWS Analogue | Use Case |
|---|---|---|
| **Single Database** | RDS single instance | One DB, one workload |
| **Elastic Pool** | (no direct AWS parallel) | Many DBs sharing compute — SaaS multi-tenant |
| **Managed Instance** | RDS Custom | Near-100% SQL Server compat, lift-and-shift |

**Elastic Pool is unique** — you pool compute (DTUs or vCores) across many databases with varying load patterns. Great for SaaS with hundreds of tenant databases where individual load is unpredictable. AWS has no direct equivalent.

**2. Purchasing Models: DTU vs vCore**

- **DTU (Database Transaction Unit)** — bundled abstract unit combining CPU, IOPS, memory. Simpler pricing, harder to tune.
- **vCore** — separate CPU and storage. Modern, better for performance tuning, supports Azure Hybrid Benefit.

**Always pick vCore for anything new.** DTU is legacy.

**3. Service Tiers (Performance/HA)**

- **Basic / Standard / Premium** — DTU model tiers
- **General Purpose** — vCore, standard workloads, single instance with fast failover
- **Business Critical** — vCore, HA with local SSD, sync replicas, read replicas
- **Hyperscale** — vCore, up to 100TB, near-instant point-in-time restore, unlimited read scale-out

Business Critical ≈ AWS RDS Multi-AZ. Hyperscale ≈ Aurora — the "Aurora killer" AWS engineers should evaluate.

**4. Built-in Features That Cost Extra on AWS**
- Automated backups included (35-day retention)
- Automatic tuning (index recommendations, auto-apply)
- Advanced Threat Protection (SQL injection detection)
- Data Discovery & Classification
- Auditing to Log Analytics

Some of these need extra config on RDS or third-party tools.

## Real Configuration

```bash
# Create a logical server (like RDS instance parent)
az sql server create \
  --name sql-prod-aakash \
  --resource-group rg-prod-db-eastus \
  --location eastus \
  --admin-user sqladmin \
  --admin-password 'StrongPassword@2026'

# Configure firewall — allow Azure services + your IP
az sql server firewall-rule create \
  --resource-group rg-prod-db-eastus \
  --server sql-prod-aakash \
  --name AllowAllAzureServices \
  --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
# (0.0.0.0/0.0.0.0 is the special value for "allow Azure services")

az sql server firewall-rule create \
  --resource-group rg-prod-db-eastus \
  --server sql-prod-aakash \
  --name AllowMyIP \
  --start-ip-address 203.0.113.42 --end-ip-address 203.0.113.42

# Create the database (General Purpose, 2 vCore)
az sql db create \
  --resource-group rg-prod-db-eastus \
  --server sql-prod-aakash \
  --name db-orders \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --zone-redundant false

# Get connection string
az sql db show-connection-string \
  --server sql-prod-aakash \
  --name db-orders \
  --client sqlcmd
```

## Production Use Case

**E-commerce order database, production tier:**

```
Azure SQL Server: sql-prod-orders (Entra ID auth + SQL auth disabled)
├── Database: db-orders
│   ├── Tier: Business Critical, Gen5, 4 vCore, 500GB
│   ├── Zone-redundant: Yes (3-AZ replication)
│   ├── Backup retention: 35 days short-term + 12 months long-term
│   ├── Automatic tuning: enabled (create/drop indexes, force last-good plan)
│   ├── Threat Protection: enabled, alerts to Security Center
│   └── Read replica: 1 (for reporting workload)
├── Networking: Private Endpoint in snet-db (no public access)
├── Auditing: enabled to Log Analytics workspace
└── Access: Managed Identity from app tier, no passwords
```

Reads and writes from app tier via managed identity. Reporting queries route to read replica via connection string with `ApplicationIntent=ReadOnly`.

## Console Path

- Portal → **SQL databases** (search) OR **SQL servers** for the container
- Database detail:
  - **Overview** → connection strings, resource utilization
  - **Query editor (preview)** → run SQL directly in portal
  - **Backup and restore** → point-in-time restore
  - **Automatic tuning** → toggle features
  - **Networking** → firewall rules, private endpoints
  - **Security** → auditing, threat protection, data classification
  - **Replicas** → geo-replication and read scale-out

## Memory Hint

- **Single DB** → one workload
- **Elastic Pool** → SaaS multi-tenant (unique to Azure)
- **Managed Instance** → lift-and-shift from on-prem SQL Server
- **Business Critical** = Multi-AZ (RDS)
- **Hyperscale** = Aurora equivalent
- **Always vCore, never DTU** (for new deployments)

## Interview Q&A — Azure SQL

**Q: Compare Azure SQL Database Business Critical tier to AWS RDS Multi-AZ.**

> "Similar goal, different mechanics.
>
> RDS Multi-AZ maintains a synchronous standby in a different AZ. Standby doesn't serve traffic — pure passive backup. Automatic failover in ~60 seconds.
>
> Azure SQL Business Critical maintains multiple synchronous replicas — one primary + three secondaries — using an Always On Availability Group under the hood. Local SSD for storage, sub-millisecond latency. Failover under 30 seconds typically.
>
> Two Business Critical wins over Multi-AZ:
>
> One: read scale-out is included. Attach `ApplicationIntent=ReadOnly` to your connection string and reads route to a secondary replica automatically. Same feature costs extra on RDS via Read Replicas.
>
> Two: local SSD storage. Business Critical is designed for high-transaction OLTP with sub-ms latency. RDS uses network-attached EBS — good but higher latency floor.
>
> Where RDS wins: cross-region read replicas are easier to configure. Azure SQL supports geo-replication but the setup is slightly clunkier.
>
> If I'm designing a new OLTP database and cost isn't the primary constraint, I'd pick Business Critical over Multi-AZ RDS for the sub-ms latency and free read replica included. If I'm cost-sensitive, General Purpose tier is enough — similar architecture to Multi-AZ RDS."

**Q: A team has 200 SaaS tenants, each with their own database, unpredictable load per tenant. How would you cost-optimize on Azure?**

> "Classic Elastic Pool use case. Azure's answer to this exact problem — AWS doesn't have a native equivalent, teams usually use RDS multi-tenant with schema-per-tenant or database-per-tenant on shared instances.
>
> Elastic Pool: allocate a pool of vCore and storage capacity, then share it across all 200 databases. When Tenant A is idle, its capacity is free for Tenant B who's active. Because tenants rarely all peak simultaneously, you can support 200 databases with maybe 30% of the capacity of provisioning each individually.
>
> Design:
> - One Azure SQL Server, one Elastic Pool with 16 vCore Gen5, 500GB pooled storage
> - All 200 tenant databases attached to this pool
> - Per-database min and max vCore caps so noisy neighbors can't starve others
> - Autoscale the pool based on DTU/vCore utilization
>
> Migration path: script move databases into the pool one at a time, verify performance, iterate on pool sizing.
>
> Reporting story: for cross-tenant analytics you'd typically ETL to a separate warehouse — Synapse or Fabric — rather than querying the pool databases. Pool is optimized for the OLTP workload.
>
> This kind of design question is where knowing an Azure-native feature (Elastic Pool) puts you ahead of engineers who only know AWS patterns."
