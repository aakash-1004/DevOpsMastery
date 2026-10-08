---
title: DynamoDB — NoSQL
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
order: 11
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-10-RDS]]"
next: "[[AWS-12-Lambda]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# DynamoDB — NoSQL

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-10-RDS|RDS — Relational Database Service]] · → [[AWS-12-Lambda|Lambda — Serverless Compute]]
<!-- /nav -->

## Concept

**DynamoDB** is AWS's managed **NoSQL** database — designed for massive scale, single-digit millisecond latency, and serverless operation.

**NoSQL means:** flexible schema, non-tabular data structure, ideal for unstructured or semi-structured data.

## Data Model

```
Data stored as items in tables
Each item = JSON-like document (key-value pairs)

Example table: users
    ├── Item 1: {name: "Raju", age: 25}
    ├── Item 2: {name: "Sham", age: 28}
    └── Item 3: {name: "Baburao", age: 45, city: "Mumbai"}

Notice: Item 3 has a 'city' field that others don't have.
That's schema flexibility — no fixed columns like SQL.
```

## Key Concepts

```
Table
    ├── Container for items (like a "table" in SQL, but flexible schema)
    └── Requires primary key definition

Primary Key
    ├── Simple: single attribute (e.g., user_id)
    ├── Composite: partition key + sort key (e.g., user_id + timestamp)
    └── Uniquely identifies each item

Item
    ├── One record in the table
    ├── Collection of attributes
    └── Max 400KB per item

Attribute
    ├── A field on an item
    └── Types: String, Number, Binary, Boolean, List, Map, Set
```

## Capacity Modes

```
On-Demand (Pay-per-request)
    ├── No capacity planning needed
    ├── Instantly scales to any load
    ├── Pay for actual reads/writes
    └── Best for: variable/unpredictable workloads

Provisioned Capacity
    ├── Pre-allocate Read/Write Capacity Units (RCU/WCU)
    ├── Auto-scaling available
    ├── Cheaper for predictable workloads
    └── Best for: steady, predictable traffic
```

**Free tier:** 25 GB storage + 25 RCU + 25 WCU (handles ~200 million requests/month).

## Serverless Benefits

```
No Server Management
    ├── No provisioning, no OS, no patching
    └── AWS handles everything

Automatic Scaling
    ├── Scales instantly based on demand
    └── Handles traffic spikes without config

Zero Downtime
    ├── No maintenance windows
    └── Continuous availability

Pay-as-You-Go
    ├── Only pay for actual usage
    └── Scales to zero when idle (no charge)

Idle Cost Savings
    └── Empty tables cost nothing
```

## Use Cases

**Ideal for:**

- Mobile applications (real-time sync)
- Web apps with high traffic
- Gaming (leaderboards, player state)
- IoT (sensor data)
- Ad tech (user profiles, tracking)
- Real-time analytics
- Session storage

**Not ideal for:**

- Complex JOINs (NoSQL doesn't do JOINs well)
- Ad-hoc queries on non-key attributes (need to design keys correctly)
- Strong ACID transactions across many tables

## DynamoDB Accelerator (DAX)

**In-memory caching layer for DynamoDB.**

```
DAX offers:
    ├── Microsecond latency (10x faster than DynamoDB)
    ├── Fully managed cache
    ├── Deployed in multiple AZs for HA
    └── Transparent to your app (same API)

Only for DynamoDB (not other databases like ElastiCache is)
```

**When to use:** Read-heavy workloads where you're hitting DynamoDB's already-fast performance limits.

## DynamoDB Global Tables

**Multi-region replication with eventual consistency.**

```
Region 1 (Mumbai)  ←→  Region 2 (US-East)  ←→  Region 3 (EU-West)

Writes in any region replicate to all others.
Users read/write in nearest region for low latency.
```

**Use case:** Global apps requiring low latency worldwide.

## Practical Setup

**Scenario:** EC2 with Docker running a Node.js app that reads/writes to DynamoDB.

**Steps:**

```
1. Create DynamoDB Table
   Name: 'Contacts'
   Partition key: 'id' (Type: String)
   Capacity: On-Demand or Provisioned

2. Create EC2 Instance
   Install Docker, pull app image

3. Create IAM Access Keys
   Access Key + Secret Key for app authentication
   BETTER: use IAM Role attached to EC2 (no keys!)

4. Run Container with Credentials
   docker run --rm -d -p 80:3000 --name node-dynamo-app \
     -e AWS_REGION=ap-south-1 \
     -e AWS_ACCESS_KEY_ID=xxx \
     -e AWS_SECRET_ACCESS_KEY=yyy \
     philippaul/node-dynamodb-demo
```

**Production version (no access keys):**

```
Attach IAM Role to EC2 with DynamoDB permissions
App uses AWS SDK — auto-detects role credentials
No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY in env
```

## DynamoDB vs RDS — Quick Comparison

```
Feature           RDS                    DynamoDB
─────────────────────────────────────────────────────────
Type              Relational (SQL)       NoSQL (Key-value)
Schema            Fixed, rigid           Flexible
Scaling           Vertical + read replicas Horizontal (auto)
JOINs             Yes                    No (denormalize)
ACID              Full                   Limited (single item)
Complexity        More setup             Fully managed
Best for          Structured data,       Unstructured, high scale
                  complex queries        real-time
Latency           Millisecond            Single-digit ms
                                         (DAX: microsecond)
```

## Memory Hints

- **DynamoDB** = "NoSQL on autopilot"
- **Primary Key** = "unique fingerprint for each item"
- **On-Demand** = "pay only when used" (like Uber pricing)
- **Provisioned** = "reserved parking spot" (pay for allocated capacity)
- **DAX** = "turbo boost for DynamoDB"
- **Global Tables** = "same data everywhere in the world"

## Interview Q&A — DynamoDB

**Q: When would you use DynamoDB over RDS?**

> "DynamoDB when you need massive scale, unpredictable traffic, or don't want to manage anything.
>
> Perfect use cases: mobile app backends where user count can spike from thousands to millions overnight. Gaming leaderboards with millions of concurrent players. IoT platforms ingesting sensor data at high volume. Session storage for web apps. Real-time analytics dashboards.
>
> RDS when you have structured, relational data with complex queries — orders, customers, products with foreign key relationships. Financial applications requiring full ACID transactions across multiple tables. Legacy apps designed for SQL.
>
> Key trade-off: DynamoDB requires careful key design upfront. You can't do ad-hoc queries like 'find all orders over $100 in the last month' unless you designed your keys for that query. RDS is more flexible for evolving query patterns.
>
> Modern approach: use both. DynamoDB for high-volume operational data (sessions, real-time state), RDS for structured business data (orders, products, users). Some architectures use DynamoDB streams to sync data into RDS or Redshift for analytics."

**Q: Explain DynamoDB's capacity modes.**

> "Two modes: On-Demand and Provisioned.
>
> On-Demand: pay per read and write request. No capacity planning. DynamoDB scales instantly to any load. Best for new applications, unpredictable traffic, or dev/test environments. Downside: more expensive per request than provisioned.
>
> Provisioned: you pre-allocate Read Capacity Units and Write Capacity Units. DynamoDB reserves that capacity. Cheaper if your traffic is predictable and steady. Comes with Auto Scaling — DynamoDB automatically adjusts provisioned capacity based on utilization. Good for production workloads with known baselines.
>
> Decision framework: start with On-Demand for new apps to avoid over/under-provisioning. Monitor for 2-4 weeks to understand patterns. Switch to Provisioned with Auto Scaling once you know your baseline — typically 40-60% cheaper for steady workloads.
>
> You can switch modes every 24 hours if patterns change."
