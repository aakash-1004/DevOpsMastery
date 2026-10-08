---
title: "Cloud Functions & Cloud Run"
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
order: 11
up: "[[GCP-MOC]]"
prev: "[[GCP-09-Firestore-Bigtable-Spanner]]"
next: "[[GCP-11-IaC-Deployment-Manager-Config-Connector-Terraform]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Cloud Functions & Cloud Run

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-09-Firestore-Bigtable-Spanner|Firestore, Bigtable, Spanner]] · → [[GCP-11-IaC-Deployment-Manager-Config-Connector-Terraform|IaC — Deployment Manager, Config Connector, Terraform]]
<!-- /nav -->

GCP has **two distinct serverless compute products**, and understanding both is critical for interviews because **Cloud Run has no direct equivalent in AWS or Azure** at the same level of maturity.

---

## 10a. Cloud Functions

**Concept:** Event-driven, code-based serverless. Deploy a function, GCP runs it in response to triggers.

### AWS Equivalent

**Cloud Functions = AWS Lambda**

### Azure Equivalent

**Cloud Functions = Azure Functions**

### Three-Way Comparison

| Feature | AWS Lambda | Azure Functions | Cloud Functions (2nd gen) |
|---|---|---|---|
| Trigger types | Broad (S3, SQS, API Gateway, etc.) | Broad (Blob, Cosmos, Queue, HTTP) | HTTP + Pub/Sub + Storage + Eventarc |
| Cold start | Fast for JS/Python, slow for JVM | Fast for JS/Python, slow for JVM | Similar |
| Max duration | 15 minutes | 10 min Consumption / unlimited Premium | 60 min (2nd gen) |
| Memory | Up to 10GB | Up to 14GB (Premium) | Up to 32GB (2nd gen) |
| Pricing | Requests + GB-second | Requests + GB-second | Requests + GB-second |

### Cloud Functions 1st Gen vs 2nd Gen

Two versions exist:
- **1st gen** — original, simpler, fewer features. Legacy.
- **2nd gen** — built on Cloud Run under the hood, more features (longer duration, more concurrency per instance, better scaling).

**Always use 2nd gen for new work.** It's essentially Cloud Run with function-style deployment ergonomics.

### Real Configuration

```bash
# Deploy a Python HTTP function (2nd gen)
gcloud functions deploy hello-world \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=hello_world \
  --trigger-http \
  --allow-unauthenticated \
  --memory=512Mi

# Deploy triggered by Cloud Storage event
gcloud functions deploy process-upload \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_upload \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=myapp-prod-uploads" \
  --service-account=processing-sa@myapp-prod.iam.gserviceaccount.com
```

---

## 10b. Cloud Run

**Concept:** Fully managed serverless containers. Deploy a container image, GCP runs it in response to HTTP requests, auto-scales to zero when idle, scales up to thousands of instances under load.

### AWS Equivalent

**Cloud Run ≈ AWS App Runner** (approximately, but Cloud Run is more mature and flexible)

Also related to AWS Fargate for containers, but Fargate is minimum-instance-based (not scale-to-zero).

### Azure Equivalent

**Cloud Run ≈ Azure Container Apps** (approximately, but Cloud Run is simpler and more mature)

### Why Cloud Run Is Genuinely Different

Cloud Run's pricing model, developer experience, and scaling behavior are best-in-class among managed container services. Key differentiators:

**1. True scale-to-zero.** Idle service costs $0. You pay per request + CPU-seconds during requests. AWS App Runner has some scale-to-zero support but with cold-start caveats. Container Apps supports scale-to-zero but is more complex to configure.

**2. Request-based pricing.** Cloud Run bills per 100ms of CPU used per request. Not per instance-hour. For sporadic traffic, this is dramatically cheaper than running containers 24/7.

**3. Any container works.** As long as your container listens on the port from `$PORT` env var, it runs. No AWS Lambda base image requirements. No custom runtimes. Docker image, deploy, done.

**4. Concurrency per instance is configurable.** By default one Cloud Run instance handles 80 concurrent requests. Not 1 request per instance like Lambda. This makes containerized apps much more efficient than function-per-request.

**5. Sidecar containers supported.** You can deploy multiple containers per Cloud Run service — one main container plus sidecars (like a Kubernetes pod). Useful for auth proxies, logging agents.

### Cloud Run Modes

Two flavors:
- **Cloud Run services** — long-lived, HTTP-triggered, request-driven scaling
- **Cloud Run jobs** — batch execution, run once, exit (like Kubernetes Jobs)

### Real Configuration — Cloud Run Service

```bash
# Build and push container to Artifact Registry
gcloud builds submit --tag us-central1-docker.pkg.dev/myapp-prod/repo/api:v1

# Deploy to Cloud Run
gcloud run deploy api \
  --image=us-central1-docker.pkg.dev/myapp-prod/repo/api:v1 \
  --region=us-central1 \
  --platform=managed \
  --service-account=api-sa@myapp-prod.iam.gserviceaccount.com \
  --allow-unauthenticated \
  --port=8080 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=100 \
  --concurrency=80 \
  --timeout=60s \
  --set-env-vars="ENV=production,DB_HOST=10.0.20.5" \
  --vpc-connector=projects/myapp-prod/locations/us-central1/connectors/serverless-vpc

# Update to new version (blue-green style via revisions)
gcloud run deploy api \
  --image=us-central1-docker.pkg.dev/myapp-prod/repo/api:v2 \
  --region=us-central1 \
  --no-traffic  # Deploy new revision without sending traffic

# Gradually shift traffic
gcloud run services update-traffic api \
  --region=us-central1 \
  --to-revisions=api-v2=20,api-v1=80

# Full cutover
gcloud run services update-traffic api \
  --region=us-central1 \
  --to-latest
```

### Production Use Case

**Full application as Cloud Run services:**

```
Project: myapp-prod
├── Cloud Run service: web (frontend, allow-unauthenticated)
│   ├── Container: nginx serving React build
│   ├── Concurrency: 250 (mostly static file serving)
│   ├── Min instances: 1 (always warm for fast first response)
│   └── Custom domain: www.aakashrao.dev via domain mapping
│
├── Cloud Run service: api (backend, require auth)
│   ├── Container: FastAPI app
│   ├── Concurrency: 80 (mix of CPU-heavy and I/O)
│   ├── Min instances: 0 (scale to zero when idle)
│   ├── VPC connector: for Cloud SQL private IP access
│   └── Service Account: api-sa with Cloud SQL + Storage roles
│
├── Cloud Run job: nightly-etl
│   ├── Runs at 03:00 UTC via Cloud Scheduler
│   ├── Container: dbt runner
│   └── Timeout: 1 hour, max 3 retries
│
└── Ingress: Global External Application LB routes:
    ├── /*  → web service (via serverless NEG)
    ├── /api/*  → api service (via serverless NEG)
    └── Cloud CDN caches /static/* on the LB
```

Zero infrastructure to manage. No Kubernetes cluster. No VMs. No autoscaling policies. Deploy, done.

### Cloud Run vs Cloud Functions — Which to Pick?

| Consideration | Cloud Functions | Cloud Run |
|---|---|---|
| Language flexibility | Fixed runtimes (Python 3.12, Node 20, etc.) | Any container |
| Portability | GCP-only | Container-native, portable |
| Complexity | Low (single file works) | Slightly higher (Dockerfile needed) |
| Concurrency per instance | 1 (each request = new instance) | Configurable (up to 1000) |
| Max duration | 60 min (2nd gen) | 60 min (services), unlimited (jobs) |
| Custom system deps | Limited | Unlimited (any container) |

**Rule of thumb:**
- **Event-driven glue code, simple HTTP handlers** → Cloud Functions
- **Anything containerized, any real application** → Cloud Run
- **When in doubt, Cloud Run** — it's more flexible and 2nd gen Functions is Cloud Run under the hood anyway

### Console Path

- Console → **Cloud Run** → **Services** or **Jobs**
- Service detail: **Revisions**, **Metrics**, **Logs**, **Triggers**, **Networking**, **Security**
- Traffic splitting: **Manage Traffic** button on service detail

### Memory Hint

- **Cloud Functions 2nd gen** = Lambda equivalent (fixed runtimes)
- **Cloud Run** = "AWS App Runner + Azure Container Apps but done right"
- **Scale to zero** by default (cost efficiency)
- **Request-based pricing** — pay per CPU used, not per instance-hour
- **Concurrency per instance** = configurable (game-changer vs Lambda's 1-per-instance)
- **Any container works** — Dockerfile is enough

### Interview Q&A — Cloud Run

**Q: A team is running dockerized microservices on ECS Fargate. Would you migrate to Cloud Run? Why or why not?**

> "Depends on their access patterns and integration needs, but Cloud Run is often a strong win.
>
> When Cloud Run beats Fargate:
>
> First, cost for spiky or sporadic workloads. Fargate has minimum instance count and charges per CPU/RAM-hour of allocated capacity. Cloud Run bills per 100ms of CPU actually used per request AND supports scale-to-zero. A service with 100 requests/day pays essentially $0 on Cloud Run and $50-100/month on Fargate.
>
> Second, developer experience for simple deployments. `gcloud run deploy --image=...` and you're live. No task definitions, service definitions, ALB target groups, security groups. One command.
>
> Third, concurrency per instance. Cloud Run defaults to 80 concurrent requests per instance. Fargate is one request handled per instance thread. For I/O-bound apps, Cloud Run is dramatically more efficient with fewer instances.
>
> When Fargate might win:
>
> AWS-native integrations. If the service depends on other AWS services with tight IAM integration, staying on Fargate reduces cross-cloud complexity.
>
> Long-running or high-concurrency workloads at steady load. Fargate's pricing per allocated capacity can be cheaper than Cloud Run's per-request billing if you're at 100% utilization 24/7.
>
> Specific networking needs. Fargate integrates deeply with VPC networking, service discovery, service mesh. Cloud Run has serverless VPC connectors and Direct VPC egress but the model is different.
>
> Practical recommendation: for greenfield services on a fresh GCP project, Cloud Run is the default. For migrating existing Fargate services with AWS integrations, evaluate case-by-case. The Cloud Run developer experience is worth optimizing for even if there's some transition cost."

**Q: Compare Cloud Run to Azure Container Apps and AWS App Runner.**

> "All three are managed container services with auto-scaling. The nuances matter.
>
> Cloud Run: the most mature and simplest. Deploy a container, done. Scale-to-zero works, request-based pricing works, concurrency-per-instance is configurable. Been GA since 2019 and battle-tested.
>
> Azure Container Apps: newer (2022 GA), built on Kubernetes + KEDA under the hood. More features exposed (Dapr, KEDA scaling on any metric, revisions with traffic splitting). Also more concepts to learn — environments, replicas, scale rules. If you need advanced scaling on custom metrics like queue depth or Kafka lag, Container Apps has richer primitives.
>
> AWS App Runner: newest of the three (2021 GA), simplest of the three. Deploy from container registry or from source code repo. Scales based on requests. Less feature-rich than Cloud Run or Container Apps — no traffic splitting, less configurability. Good for simple web services, less good for anything complex.
>
> Rough ranking by developer experience: Cloud Run > App Runner > Container Apps for simple cases. Cloud Run > Container Apps > App Runner for advanced use cases.
>
> Portfolio tip: build a demo project on Cloud Run and mention it in interviews as evidence of cross-cloud fluency. It's a service that's genuinely fun to use and easy to demonstrate."
