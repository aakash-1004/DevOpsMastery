---
title: RDS — Relational Database Service
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
order: 10
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-09-S3]]"
next: "[[AWS-11-DynamoDB]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# RDS — Relational Database Service

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-09-S3|S3 — Simple Storage Service]] · → [[AWS-11-DynamoDB|DynamoDB — NoSQL]]
<!-- /nav -->

## Concept

**AWS RDS** is a **managed relational database service** that handles administrative tasks: backups, patching, monitoring, scaling.

**What "managed" means:**

- You don't SSH into the DB server
- AWS handles OS updates, DB engine patches
- Automated backups
- Multi-AZ failover
- Read replicas
- Monitoring

You just define: engine, instance size, storage, and connection settings. AWS does the rest.

## Supported Database Engines

```
Amazon Aurora    → AWS-native MySQL/PostgreSQL compatible (fastest)
MySQL            → Community MySQL, managed
PostgreSQL       → Community PostgreSQL, managed
MariaDB          → MySQL fork, managed
Oracle           → Enterprise DB, license required
SQL Server       → Microsoft, license required or BYOL
```

## Amazon Aurora — The Star

Aurora is AWS's proprietary database engine, MySQL-and-PostgreSQL-compatible.

```
Aurora offers:
  ├── Up to 5x throughput of MySQL Community Edition
  ├── Up to 3x throughput of PostgreSQL
  ├── Up to 128 TB of autoscaling SSD storage
  ├── Six-way replication across three AZs (2 copies per AZ)
  ├── Up to 15 read replicas with < 10ms lag
  └── Automatic monitoring with failover
```

**Why so fast:** Aurora separates compute from storage. Storage is a distributed, self-healing service across multiple AZs. Compute instances are stateless — can be replaced instantly without data loss.

## Key Benefits of RDS

```
High Availability
    ├── Multi-AZ deployments with synchronous replication
    └── Automatic failover in ~60 seconds

Scaling
    ├── Vertical: change instance type (t3.medium → r5.large)
    ├── Horizontal: add read replicas for read-heavy workloads
    └── Storage: auto-scaling storage

Automated Backups
    ├── Daily snapshots (default retention 7 days, max 35)
    ├── Continuous transaction logs — point-in-time recovery
    └── Can restore to any second in the retention window

Read Replicas
    ├── Async replication to separate instances
    ├── Offload read traffic from primary
    └── Can be in different regions

Multi-AZ for Disaster Recovery
    ├── Synchronous standby in different AZ
    ├── Automatic failover
    └── No data loss

Cost-Effectiveness
    ├── Cheaper than self-managed DBs on EC2
    ├── No DBA needed for routine ops
    └── Free tier available
```

## Multi-AZ vs Read Replicas

**These solve DIFFERENT problems — critical interview topic.**

```
Multi-AZ (High Availability)
    ├── Synchronous replication to standby in another AZ
    ├── Standby does NOT serve traffic — passive backup
    ├── Automatic failover on primary failure (~60 seconds)
    ├── DNS endpoint updates automatically
    └── PURPOSE: minimize downtime, disaster recovery

Read Replicas (Read Scaling)
    ├── Asynchronous replication to separate instances
    ├── Replicas DO serve read traffic
    ├── Can be in same region or cross-region
    ├── Manual promotion to primary if needed
    └── PURPOSE: scale read performance
```

**Real world:** Use both together.

- Multi-AZ for the primary → automatic failover
- Read replicas for scaling read queries → analytics, reports, dashboards

## Practical Setup

**Scenario:** EC2 running Node.js app, connecting to RDS MySQL.

**Steps:**

```
1. Create RDS MySQL Instance
   ├── Use Free Tier (db.t3.micro)
   ├── Username: admin, set password (no special chars)
   ├── Public access: True (for lab access; NEVER in production)
   ├── Security group: allow port 3306 from EC2 SG
   └── After creation: note the ENDPOINT (hostname)

2. Create EC2 Instance
   ├── Amazon Linux 2
   ├── Install Docker
   └── Pull Node.js app image

3. Run App with DB Credentials as Env Vars
   docker run --rm -p 80:3000 \
     -e DB_HOST="myapp-db.xxxxx.ap-south-1.rds.amazonaws.com" \
     -e DB_USER="admin" \
     -e DB_PASSWORD="secretpassword" \
     -d philippaul/node-mysql-app:02

4. Verify Connection
   docker run -it --rm mysql:8.0 mysql -h myapp-db.xxxxx.rds.amazonaws.com -u admin -p
```

## Production Setup

**Never** make RDS publicly accessible in production. Correct setup:

```
Public Subnets:
    ├── ALB (internet-facing)
    └── NAT Gateway

Private App Subnets:
    ├── EC2 instances (behind ALB)
    └── Connect to RDS via private IP

Private DB Subnets:
    ├── RDS Multi-AZ (primary in AZ-a, standby in AZ-b)
    ├── Security group: only allow 3306 from App SG
    └── NOT reachable from internet
```

## RDS Storage Auto-Scaling

**Setting:** Storage auto-scaling threshold — e.g., 20% free space.

**Behavior:** When free space drops below 20%, RDS automatically increases storage (up to max limit you set). No downtime. Prevents "database full" incidents.

## Common Use Cases

**Web Applications:** Structured data with ACID transactions — user profiles, orders, sessions. **E-commerce:** Inventory, customer data, orders — need strong consistency. **Business Applications:** ERP, CRM, finance apps — need data integrity.

## Memory Hints

- **RDS** = "database as a service" (AWS handles the boring stuff)
- **Aurora** = "the Ferrari version" (AWS-optimized MySQL/PostgreSQL)
- **Multi-AZ** = "hot standby in another AZ" (for HA, doesn't serve traffic)
- **Read Replica** = "clone that answers questions" (for read scaling)
- **Endpoint** = the DNS name that points to your DB (survives failover)

## Interview Q&A — RDS

**Q: What's the difference between Multi-AZ and Read Replicas?**

> "Both involve replication but solve different problems.
>
> Multi-AZ is for high availability. AWS creates a synchronous standby replica in a different AZ. The standby doesn't serve traffic — it's a hot backup. If the primary fails, RDS automatically fails over to the standby within about 60 seconds. Your app doesn't need to change anything because the RDS endpoint stays the same — it just points to the new primary. Purpose: minimize downtime from AZ failures, hardware failures, or planned maintenance.
>
> Read Replicas are for read scaling. AWS creates asynchronous replicas that DO serve read queries. Your app routes writes to the primary and reads to the replicas. This offloads the primary and improves overall throughput. Purpose: handle read-heavy workloads like analytics, reports, or high-traffic reads. Can be in same region for latency, or cross-region for geographic distribution.
>
> In production I use both together. Multi-AZ for the primary provides automatic disaster recovery. Two or three read replicas handle report queries and heavy read traffic without impacting transactional performance on the primary."

**Q: When would you choose Aurora over RDS MySQL?**

> "Aurora when performance, availability, and scale matter more than cost. Aurora offers up to 5x MySQL throughput, storage that auto-scales to 128TB with 6-way replication across 3 AZs, 15 read replicas with sub-10ms lag. It's the gold standard for AWS databases.
>
> RDS MySQL when you need compatibility with community MySQL features or exact version matching for a legacy app. Also when cost is a major constraint — Aurora is more expensive than RDS MySQL.
>
> For most modern applications building on AWS, Aurora is the default choice. The performance and availability benefits justify the cost difference. You get built-in features that would take significant engineering effort to replicate on RDS MySQL or self-managed.
>
> Aurora Serverless is another consideration — Aurora that scales compute automatically based on load, including scaling to zero when idle. Great for dev/test environments or unpredictable workloads."

**Q: How do you handle RDS backups and point-in-time recovery?**

> "RDS provides two backup mechanisms.
>
> Automated Backups: daily snapshots plus continuous transaction logs. Retention 1-35 days configurable. This enables point-in-time recovery — you can restore to any second within the retention window. If someone runs an `UPDATE` without a `WHERE` clause at 2:47 PM, you restore to 2:46 PM.
>
> Manual Snapshots: on-demand snapshots you take before major changes. These don't expire — persist until you delete them. Perfect for pre-migration checkpoints or long-term compliance archives.
>
> Restore process: you don't restore in-place. RDS creates a NEW database from the backup. Your app connects to the new endpoint. This preserves the original for further investigation.
>
> Cross-region: automated backups don't automatically cross regions. For DR, either configure cross-region snapshot copy, or use Aurora's global database feature which replicates to another region with under 1 second lag.
>
> Production practice: automated backups with 7-day retention plus weekly manual snapshots for a rolling 90-day archive. Test restore process quarterly — untested backups aren't backups."
