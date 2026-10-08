---
title: "Firestore, Bigtable, Spanner"
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
order: 10
up: "[[GCP-MOC]]"
prev: "[[GCP-08-Cloud-SQL]]"
next: "[[GCP-10-Cloud-Functions-and-Cloud-Run]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Firestore, Bigtable, Spanner

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-08-Cloud-SQL|Cloud SQL]] · → [[GCP-10-Cloud-Functions-and-Cloud-Run|Cloud Functions & Cloud Run]]
<!-- /nav -->

GCP splits what AWS calls "DynamoDB" into **three distinct products**. Each solves different problems. This section is where GCP's data offerings genuinely diverge from AWS and Azure — Spanner especially has no direct equivalent.

## The Three Products at a Glance

| Product | Type | Use Case | AWS Analogue | Azure Analogue |
|---|---|---|---|---|
| **Firestore** | Document DB | Mobile/web apps, small-medium scale | DynamoDB (basic) | Cosmos DB (NoSQL API) |
| **Bigtable** | Wide-column | Massive time-series, IoT, analytics | DynamoDB (scale) / Keyspaces | Cosmos DB (Cassandra API) |
| **Spanner** | Distributed SQL | Global, strongly-consistent SQL | Aurora Global / DynamoDB Global Tables | Cosmos DB (with strong consistency) |

---

## 9a. Firestore

**Concept:** Serverless document database, mobile/web-friendly, real-time listeners, offline sync.

**Two modes at creation time (cannot change later):**
- **Native mode** — modern, supports real-time listeners, better client SDKs
- **Datastore mode** — legacy Datastore API, backward compat

**For anything new: Native mode.**

### Key Features

- Document/collection model (like MongoDB or Cosmos DB Document API)
- Real-time listeners — clients subscribe, get pushed updates via WebSocket-like connection
- Offline persistence in mobile SDKs (queue writes offline, sync when back online)
- Serverless — no capacity provisioning
- Auto-scaling reads and writes

### AWS/Azure Comparison

DynamoDB is closer to Firestore than to Bigtable or Spanner — same "managed NoSQL for apps" niche. But DynamoDB doesn't have real-time listeners; you'd add DynamoDB Streams + Lambda + WebSocket API Gateway to fake it.

Azure Cosmos DB (SQL API) is Firestore's closest Azure equivalent, but Cosmos also has WebSocket-like Change Feed which requires more plumbing than Firestore's built-in listeners.

Firestore's real-time story is genuinely differentiated for mobile/web apps.

### Real Configuration

```bash
# Enable Firestore in Native mode (one-time per project)
gcloud firestore databases create --location=us-central1

# Firestore is typically used via SDK, not gcloud commands
# Example writes via curl (using access token):
TOKEN=$(gcloud auth application-default print-access-token)
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"fields":{"name":{"stringValue":"Aakash"},"role":{"stringValue":"DevOps"}}}' \
  "https://firestore.googleapis.com/v1/projects/myapp-prod/databases/(default)/documents/users/user123"
```

### Interview Q&A — Firestore

**Q: A team is building a mobile app with a chat feature. Firestore, DynamoDB, or Cosmos DB?**

> "Firestore, unambiguously. Real-time listeners are built in — chat clients subscribe to a message collection and receive updates as they happen with no polling and no additional infrastructure. Offline persistence handles poor mobile connectivity gracefully.
>
> DynamoDB requires you to build a real-time layer on top: DynamoDB Streams → Lambda → API Gateway WebSockets → clients. More moving parts, more failure modes, more code.
>
> Cosmos DB has Change Feed which is similar to Firestore under the hood, but you still need to build a WebSocket layer on top for client push. Not as turnkey.
>
> Firestore's cost model favors this use case too — you pay per document read/write plus network bandwidth. For chat apps with moderate message rates, cost is predictable and low.
>
> Where Firestore falls short: complex queries. It supports basic filters and ordering but no JOINs, no aggregate queries beyond count. If your data model needs SQL-like flexibility, look elsewhere. But for chat, notification feeds, real-time dashboards — Firestore is the shortest path from idea to production."

---

## 9b. Bigtable

**Concept:** Petabyte-scale wide-column NoSQL, HBase-compatible API. For massive time-series, IoT sensor data, analytics workloads with high throughput.

### Key Features

- Petabyte scale, single-digit millisecond latency at that scale
- HBase-compatible API (Java clients, HBase shell all work)
- Cluster-based — you provision nodes (unlike serverless Firestore)
- Multi-cluster replication across regions (async or global)
- Integrates natively with Dataflow, BigQuery, Dataproc for analytics

### AWS/Azure Comparison

Closest AWS equivalent: DynamoDB at massive scale (but different API and data model). AWS Keyspaces (Managed Cassandra) is architecturally closer.
Closest Azure equivalent: Cosmos DB Cassandra API.

**Bigtable is the go-to for genuine massive-scale time-series and analytics workloads.** Google uses it internally for search indexing, Gmail, Ads — it's battle-tested at their scale.

### When to Choose Bigtable

- Time-series at scale (financial ticks, IoT sensors, monitoring metrics)
- Ad-tech / analytics workloads with billions of rows
- Existing HBase applications migrating to cloud (drop-in compat)

**When NOT to choose Bigtable:**
- Small datasets (below ~1TB) — expensive overhead
- Transactional workloads needing SQL semantics — use Spanner
- Simple app databases — use Firestore

### Real Configuration

```bash
# Create a Bigtable instance
gcloud bigtable instances create prod-metrics \
  --display-name="Production Metrics" \
  --cluster-config=id=prod-metrics-c1,zone=us-central1-a,nodes=3 \
  --cluster-config=id=prod-metrics-c2,zone=us-east1-a,nodes=3 \
  --instance-type=PRODUCTION

# Create a table
cbt -project=myapp-prod -instance=prod-metrics createtable events \
  "families=cf1:maxversions=1"

# Write via cbt (Bigtable's admin CLI)
cbt -project=myapp-prod -instance=prod-metrics set events row1 cf1:count=42
```

---

## 9c. Spanner

**Concept:** Globally distributed, horizontally scalable, strongly consistent SQL database. **No other major cloud has an equivalent.** This is arguably GCP's biggest technical differentiator.

### Key Features

- SQL interface (Postgres and GoogleSQL dialects)
- ACID transactions across globally distributed data
- 99.999% availability SLA (five nines)
- Horizontal scaling — add nodes, capacity increases linearly
- Automatic sharding, no manual partitioning
- Uses Google's TrueTime API (atomic clock + GPS) to achieve external consistency across regions

### The Uniqueness

The CAP theorem says you can't have Consistency + Availability during a network Partition. Spanner effectively achieves both because Google's infrastructure has such low probability of partition that they can guarantee both under normal conditions.

No AWS or Azure product does this. AWS Aurora Global has async cross-region replication (eventually consistent). Cosmos DB has multi-region writes but with different consistency trade-offs. Spanner is genuinely different — strongly consistent globally, at scale.

### AWS/Azure Comparison

| Feature | AWS Aurora Global | Azure Cosmos DB (Strong) | GCP Spanner |
|---|---|---|---|
| Global writes | No (single writer region) | Yes (limited by consistency) | Yes, fully consistent |
| Cross-region consistency | Eventual | Bounded staleness or strong (single region only) | Strong, external |
| SQL interface | Yes (MySQL/Postgres) | No (Cosmos SQL is limited) | Yes (Postgres or GoogleSQL) |
| Horizontal scaling | Manual sharding | Automatic (partition key) | Automatic |
| Cost | Moderate | High | High (but unique capability) |

### When to Choose Spanner

- Financial systems where global consistency matters (payments, ledgers, inventory)
- Global multi-tenant SaaS where a user in Europe and one in Asia hit the same data with strict consistency
- Migrating from a sharded MySQL/Postgres setup that's become operationally painful

**When NOT to choose Spanner:**
- Small workloads — it's expensive
- Workloads that fit comfortably in a single-region Postgres — use Cloud SQL or AlloyDB

### Real Configuration

```bash
# Create a Spanner instance
gcloud spanner instances create prod-spanner \
  --config=regional-us-central1 \
  --description="Production Spanner" \
  --nodes=1

# For multi-region:
gcloud spanner instances create prod-spanner-global \
  --config=nam-eur-asia1 \
  --nodes=3

# Create a database
gcloud spanner databases create appdb \
  --instance=prod-spanner \
  --ddl-file=schema.sql

# Query via CLI
gcloud spanner databases execute-sql appdb \
  --instance=prod-spanner \
  --sql="SELECT * FROM users LIMIT 10"
```

### Interview Q&A — When to Use Spanner

**Q: A global fintech needs strongly consistent transactions across regions. Walk through GCP vs AWS vs Azure options.**

> "This is Spanner's canonical use case. Financial transactions require ACID guarantees — no eventual consistency, no read-your-writes anomalies, no partition tolerance shortcuts.
>
> GCP Spanner: designed exactly for this. Multi-region config (say `nam-eur-asia1`), all writes are strongly consistent globally. A payment initiated in Europe is immediately visible in Asia with correct balance. SQL interface, so existing SQL knowledge transfers.
>
> AWS: Aurora Global Database gives you cross-region reads, but writes are single-region. Failover across regions is 1-2 minutes with some data loss risk. For active-active global consistency, AWS doesn't have a direct equivalent. Some teams use DynamoDB Global Tables with careful design, but you're giving up strong consistency at global scale.
>
> Azure: Cosmos DB with Strong consistency works within a single region, or with 'Bounded Staleness' cross-region. Not equivalent to Spanner's external consistency.
>
> The catch with Spanner: it's expensive. Single-region minimum is ~$1000/month. Multi-region starts around $3000/month. Only makes sense for workloads that genuinely need this capability.
>
> Practical decision: if the workload TRULY needs global strong consistency for correctness reasons (financial systems, inventory, regulatory), Spanner is worth the cost and is the only cloud service that delivers. If you can accept eventual consistency with careful application design, AWS or Azure are cheaper.
>
> Interview signal: knowing Spanner exists AND knowing when NOT to use it (which is most of the time) is the senior answer."
