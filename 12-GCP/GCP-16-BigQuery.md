---
title: BigQuery
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
order: 17
up: "[[GCP-MOC]]"
prev: "[[GCP-15-VPC-Networking]]"
next: "[[GCP-17-Three-Way-Cross-Cloud-Interview-QA]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# BigQuery

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-15-VPC-Networking|VPC Networking]] · → [[GCP-17-Three-Way-Cross-Cloud-Interview-QA|Three-Way Cross-Cloud Interview Master Q&A]]
<!-- /nav -->

> **Tags:** #gcp #bigquery #data-warehouse #flagship
> **Status:** Not core DevOps, but you can't have a serious GCP conversation without it — it shapes many GCP architectures and is the flagship service everyone associates with GCP.
> **Interview Relevance:** ⭐⭐⭐

## Concept

**BigQuery** is GCP's serverless data warehouse — a petabyte-scale columnar analytical database that separates storage and compute. You store data cheaply, run SQL queries that scan terabytes in seconds, pay per bytes-processed or reserved slots.

Key traits: no infrastructure to manage, standard SQL interface, ML models trainable via SQL (BigQuery ML), streaming inserts, integration with every GCP data service.

## AWS Equivalent

- **Redshift** — traditional cluster-based warehouse; **Redshift Serverless** is closer but still less integrated than BigQuery
- **Athena** — serverless SQL over S3, closer conceptually but not a full warehouse

## Azure Equivalent

- **Azure Synapse Analytics** (dedicated SQL pools + serverless SQL pools) — largest analog
- **Microsoft Fabric** — newer unified analytics platform including warehouse

## Key Differences from BOTH

| Aspect | AWS Redshift | Azure Synapse | GCP BigQuery |
|---|---|---|---|
| Provisioning model | Cluster (RA3/DC2) or Serverless | Dedicated SQL pool or Serverless | **Serverless-first (or reserved slots)** |
| Pricing | Node-hours or RPU-hours | DWU-hours or per-TB scanned | **Per-TB scanned or slot-hours** |
| Storage/compute separation | Yes (RA3, Serverless) | Yes | **Yes, and long-term storage auto-tiers cheaper** |
| Streaming ingest | Kinesis → COPY or streaming ingestion | Event Hubs → Synapse | **Native streaming inserts (100k rows/sec per table)** |
| ML | Redshift ML (via SageMaker) | Synapse ML | **BigQuery ML — train models in SQL, no export** |
| Federated queries | Yes (Spectrum over S3) | External tables over ADLS | **Yes, over Cloud Storage, Cloud SQL, Spanner** |

**Why BigQuery shapes GCP architectures:**
- Most analytics workloads default to it — cheap to try (1TB free query quota per month), no cluster to size, no maintenance
- It's often the "sink" for Cloud Storage, Pub/Sub, Firestore exports
- Data engineers on GCP think BigQuery-first; on AWS, it's often S3 + Athena/Redshift; on Azure, Synapse or Fabric

## Real Configuration

```bash
# Create a dataset (like a schema/database)
bq mk --dataset --location=asia-south1 my-project-956179:analytics

# Create a table from a schema file
bq mk --table analytics.events \
  event_id:STRING,user_id:STRING,event_time:TIMESTAMP,properties:JSON

# Load data from Cloud Storage
bq load --source_format=NEWLINE_DELIMITED_JSON \
  analytics.events \
  gs://my-data-bucket/events/*.json

# Run a query (dry-run to see bytes scanned before you're billed)
bq query --dry_run --use_legacy_sql=false \
  'SELECT user_id, COUNT(*) FROM analytics.events WHERE DATE(event_time) = "2026-07-15" GROUP BY user_id'

# Stream inserts (from an app)
# Handled via the BigQuery API, not gcloud — typically from a Cloud Function
# or Cloud Run service processing Pub/Sub messages.

# Set up scheduled query (like a materialized view refresh)
bq query --use_legacy_sql=false --destination_table=analytics.daily_summary \
  --schedule='every 24 hours' \
  --display_name='Daily user summary' \
  'SELECT DATE(event_time) as day, COUNT(DISTINCT user_id) as dau FROM analytics.events GROUP BY day'
```

## Production Use Case

**Scenario:** Application on GCP writes ~10M events/day to Pub/Sub. Need to make these queryable for analytics dashboards and ad-hoc analysis, cheaply.

**Approach:**

> "Pub/Sub → Dataflow (streaming) → BigQuery. Dataflow reads from Pub/Sub, does light transformation (schema validation, timestamp normalization), streams into BigQuery via the Storage Write API.
>
> Table design:
> - Partition on `event_time` (daily partitions). Queries scanning last 7 days only scan those 7 partitions.
> - Cluster on `user_id` and `event_type`. Filter queries prune to relevant clusters.
>
> Cost control:
> - Set a project-level query cost cap (`bq --maximum_bytes_billed` in scheduled queries)
> - Use `--dry_run` in the dashboarding layer before executing expensive queries
> - Partition expiration on the events table (auto-delete data older than 90 days)
>
> For dashboards: Looker Studio (free, connects natively) or Looker (heavier BI tool, also GCP-native).
>
> Cost estimate for 10M events/day: storage ~$20/mo (long-term storage after 90 days is half price), query costs entirely usage-dependent — with partitioning and clustering, most analytics dashboards should scan under 1TB/month, staying in the free tier."

## Console Path

- Console → **BigQuery** → **SQL workspace**

## Memory Hint

- **BigQuery** = "Snowflake, but Google-native and serverless from day one"
- **Pay per bytes scanned** — always dry-run before executing on production data
- **Partition + cluster** — these are your two knobs for query cost control

## Interview Q&A — BigQuery

**Q: Why does BigQuery get mentioned so often in GCP interviews when it's a data service, not core DevOps?**

> "Because GCP is genuinely the strongest cloud for analytics workloads, and BigQuery is why. Many organizations pick GCP specifically for its data stack — BigQuery + Dataflow + Pub/Sub + Vertex AI — and the compute/networking around it becomes secondary.
>
> As a DevOps engineer on GCP, you end up managing infrastructure that feeds BigQuery: Pub/Sub topics, Dataflow pipelines, Cloud Composer (managed Airflow) DAGs, IAM for who can query what, cost governance for who can spend how much on scans. Even if you don't write queries, you're the plumber. Being able to speak intelligently about BigQuery's model — partitioning, clustering, streaming inserts, reserved slots vs on-demand — signals you understand the workloads your infrastructure supports."

**Q: A team is running Redshift on AWS and considering migration to BigQuery. What's the honest trade-off?**

> "BigQuery wins on operations: no cluster to size, no vacuum, no distribution keys, no sort keys. You create a dataset, load data, query. Redshift RA3 and Serverless have closed the gap but haven't matched the zero-touch operations of BigQuery.
>
> BigQuery wins on scalability for spiky workloads: on-demand pricing means idle time costs nothing beyond storage. Redshift Serverless charges RPU-hours even when idle if configured with a base capacity.
>
> Redshift wins on predictable, high-QPS workloads: reserved-node Redshift can be cheaper than BigQuery slots for constant-load OLAP.
>
> Redshift wins on tight AWS integration: if the rest of the stack is on AWS, moving analytics off-cloud creates egress costs and cross-cloud latency.
>
> Honest recommendation: if the team is committed to a multi-cloud future or GCP-heavy, migrate. If they're deeply AWS-integrated with steady predictable analytics load, Redshift RA3 with reserved instances is probably fine and cheaper than switching. Migration is not free — SQL dialect differences, security model changes, retraining data engineers."
