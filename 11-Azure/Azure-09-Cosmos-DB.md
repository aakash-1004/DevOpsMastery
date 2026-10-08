---
title: Cosmos DB
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
order: 9
up: "[[Azure-MOC]]"
prev: "[[Azure-08-Azure-SQL-Database]]"
next: "[[Azure-10-Azure-Functions]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Cosmos DB

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-08-Azure-SQL-Database|Azure SQL Database]] · → [[Azure-10-Azure-Functions|Azure Functions]]
<!-- /nav -->

## Concept

Azure Cosmos DB is a globally distributed, multi-model NoSQL database. Multiple wire protocols supported: SQL (native), MongoDB, Cassandra, Gremlin, Table.

## AWS Equivalent

**Cosmos DB ≈ AWS DynamoDB + DocumentDB + Neptune, unified**

Closer to DynamoDB in typical usage, but Cosmos does more — it speaks multiple APIs and offers stronger global distribution primitives out of the box.

## Key Differences from AWS

**1. Multi-Model API Selection**

You pick an API at container creation time:

| API | Use Case | AWS Analogue |
|---|---|---|
| **NoSQL (SQL API)** | Default, document DB | DynamoDB |
| **MongoDB** | Compat with existing Mongo apps | DocumentDB |
| **Cassandra** | Column-family workloads | Keyspaces |
| **Gremlin** | Graph traversal | Neptune |
| **Table** | Key-value | DynamoDB in simple mode |

The API can't be changed later. Pick based on existing code — new apps almost always use NoSQL (SQL) API.

**2. Global Distribution Is First-Class**

Cosmos DB was designed for multi-region from day one. Enable geo-replication with a checkbox:
- Add any region → data replicates asynchronously
- Multi-region writes → any region accepts writes, Cosmos merges via configured conflict resolution
- Compare to DynamoDB Global Tables — similar idea but Cosmos feels more integrated and mature

**3. Five Consistency Levels (Not Just Eventual/Strong)**

Cosmos offers a **consistency spectrum**:
- Strong (like DynamoDB strongly consistent read)
- Bounded Staleness
- Session (default — reads see own writes, most apps want this)
- Consistent Prefix
- Eventual (like DynamoDB eventually consistent read)

Each has different latency/availability trade-offs. This is a favorite interview topic — "explain the consistency levels" — because it shows depth beyond just knowing the service exists.

**4. Request Units (RU/s) Instead of RCU/WCU**

Cosmos measures throughput in RUs (Request Units per second). One RU roughly = 1 KB point read cost.

- **Provisioned throughput** — pre-allocate RU/s per container or database
- **Autoscale** — set max RU/s, Cosmos scales between 10%-100% of that automatically
- **Serverless** — pay-per-request, no capacity planning, for spiky low-volume workloads

**5. Change Feed Built In**

Every Cosmos container has a change feed — an ordered log of every modification. Consume it like a Kafka topic. Enables event-driven architectures, materialized views, ETL to other systems. DynamoDB's equivalent is Streams — Cosmos change feed is more general and richer.

## Real Configuration

```bash
# Create Cosmos DB account (NoSQL API)
az cosmosdb create \
  --name cosmos-prod-aakash \
  --resource-group rg-prod-db-eastus \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --locations regionName=westeurope failoverPriority=1 isZoneRedundant=false \
  --default-consistency-level Session \
  --enable-multiple-write-locations true

# Create a database inside the account
az cosmosdb sql database create \
  --account-name cosmos-prod-aakash \
  --resource-group rg-prod-db-eastus \
  --name AppDB

# Create a container with autoscale throughput
az cosmosdb sql container create \
  --account-name cosmos-prod-aakash \
  --resource-group rg-prod-db-eastus \
  --database-name AppDB \
  --name Contacts \
  --partition-key-path "/tenantId" \
  --max-throughput 4000

# List keys (avoid using — prefer Managed Identity)
az cosmosdb keys list \
  --name cosmos-prod-aakash \
  --resource-group rg-prod-db-eastus
```

## Partition Key — Critical Design Decision

Same as DynamoDB — choose partition key carefully. Bad partition key = throttling, hot partitions, unhappy engineers.

Rules:
- High cardinality (many distinct values)
- Even distribution (no partition disproportionately large)
- Aligns with query pattern (queries should filter on partition key when possible)

Example: For a multi-tenant SaaS `Contacts` table, `/tenantId` is usually right — spreads load per tenant, most queries scope to one tenant anyway.

Bad example: `/status` where values are `active/inactive/pending` — only 3 values, poor distribution.

## Production Use Case

**Global e-commerce cart service:**

```
Cosmos DB Account: cosmos-prod-cart
├── APIs: NoSQL (SQL API)
├── Regions:
│   ├── East US (primary, zone-redundant)
│   ├── West Europe (secondary, multi-write enabled)
│   └── Southeast Asia (secondary, multi-write enabled)
├── Consistency: Session (default per-user consistency)
├── Multi-region writes: ON (each region takes writes locally)
│
├── Database: cart-db
│   └── Container: carts
│       ├── Partition key: /userId
│       ├── Throughput: Autoscale 400-40,000 RU/s
│       ├── TTL: 30 days (auto-expire abandoned carts)
│       └── Change feed → Azure Function → analytics pipeline
│
└── Access: Managed Identity (RBAC), no keys
```

User in India adds items to cart → writes to Southeast Asia region locally → replicates to other regions in seconds. User travels to Europe next day, hits West Europe region → sees their cart (session consistency guarantees this because same user).

## Console Path

- Portal → **Azure Cosmos DB** (search)
- Account detail:
  - **Data Explorer** — query and modify data in-portal
  - **Replicate data globally** — add/remove regions with a map interface
  - **Default consistency** — change consistency level
  - **Keys** — connection strings (prefer RBAC)
  - **Scale** — throughput configuration per container
  - **Change feed** — inspect / configure processors

## Memory Hint

- **Cosmos DB** = "DynamoDB Global Tables but on steroids and multi-API"
- **RU/s** = "one point-read of 1KB ≈ 1 RU"
- **Session consistency** = default, use this unless you have a specific reason not to
- **Change Feed** = built-in Kafka-like stream, free with the container
- "Pick partition key like you're picking a DynamoDB hash key — same rules apply"

## Interview Q&A — Cosmos DB

**Q: Compare Cosmos DB to DynamoDB. When would you choose Cosmos?**

> "Similar category, different strengths.
>
> Feature parity for the basics: managed NoSQL, key-value / document, elastic scaling, global distribution. Both handle massive scale.
>
> Cosmos wins on:
>
> First, multi-region writes maturity. Both support it, but Cosmos's design and conflict resolution feels more production-ready. DynamoDB Global Tables have caught up but Cosmos had a head start.
>
> Second, consistency levels. Cosmos offers five levels on a spectrum. DynamoDB gives you strong or eventual. If your app needs 'session' consistency — user always reads their own writes but doesn't care about global ordering — Cosmos gives you that natively without eventual consistency's staleness issues.
>
> Third, multi-model. If you have existing MongoDB or Cassandra code, Cosmos speaks those wire protocols. DynamoDB is DynamoDB — port your code.
>
> DynamoDB wins on:
>
> First, ecosystem integration if you're already deep in AWS. Cosmos DB with Lambda vs DynamoDB with Lambda — the AWS pairing is smoother.
>
> Second, sometimes cost. For simple key-value patterns at scale with predictable load, DynamoDB provisioned mode can be cheaper. Cosmos's flexibility comes at a price premium.
>
> In an interview: if the question is 'what would you choose,' probe requirements first. Multi-region writes needed? Session consistency important? Legacy MongoDB code to migrate? Those tilt to Cosmos. AWS-native shop with simple key-value? DynamoDB is fine."

**Q: Walk me through Cosmos DB consistency levels.**

> "Five levels, ordered from strongest to weakest guarantees.
>
> Strong: linearizable. Any read returns the most recent committed write across all regions. Highest latency, highest cost. Use when correctness beats performance — financial ledgers, inventory counters.
>
> Bounded Staleness: reads can lag writes by a defined number of versions or a time window. So you might be up to 5 seconds or 100 versions behind. Useful when you want mostly-fresh reads but can accept a small staleness bound.
>
> Session: my go-to default. Guarantees read-your-own-writes for a given client session, monotonic reads within the session, but different sessions may see slightly different orderings. Perfect for user-facing apps — the user always sees their most recent action.
>
> Consistent Prefix: never see out-of-order writes. If writes happened in order A, B, C, you might see just A or A,B or A,B,C — never A,C without B. No time bound on freshness though.
>
> Eventual: cheapest, most available, most stale. Reads may lag arbitrarily behind writes. Fine for analytics dashboards where slight delay doesn't matter.
>
> Practical rule: start with Session. Move to Strong for the specific tables that need it — usually few. Never use Eventual for user-facing reads. This is exactly the kind of nuanced answer that separates senior candidates."

**Q: You have a Cosmos container throttling with 429 errors. How do you diagnose?**

> "429 = Request Unit budget exceeded. Systematic diagnosis:
>
> First, check the metric 'Total Request Units' in Azure Monitor. See if consumption is bumping against provisioned throughput. If yes, obvious answer: increase throughput or enable autoscale.
>
> But usually the more interesting answer is uneven partition load — a 'hot partition.' Check the 'Normalized RU Consumption' metric per partition key range. If one partition is at 100% while others are at 5%, you have a hot partition. Total throughput might be fine on paper but one partition is bottlenecked.
>
> Root cause: bad partition key. Skew in the data or query pattern. Example: partition key `/status` with 3 values, 90% of writes go to `active` — that partition drowns while others idle.
>
> Fix: redesign the partition key. This is a rebuild — Cosmos can't change partition key in place. You provision a new container with a better key, dual-write during migration, backfill, cut over. Painful. That's why partition key choice is a big deal upfront.
>
> Second common cause: expensive queries scanning across many partitions. Check for cross-partition queries in the query metrics. Rewrite them to filter on partition key when possible, or add composite indexes.
>
> Third: burst traffic that momentarily exceeds provisioned RU. Enable autoscale so throughput can absorb bursts up to 10x the baseline."
