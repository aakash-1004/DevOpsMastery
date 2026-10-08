---
title: "Three-Way Cross-Cloud Interview Master Q&A"
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
order: 18
up: "[[GCP-MOC]]"
prev: "[[GCP-16-BigQuery]]"
next: "[[GCP-18-Wrap-Up-Whats-Next]]"
origin: DevOps-Mastery/12-GCP/GCP Mastery.md
---

# Three-Way Cross-Cloud Interview Master Q&A

<!-- nav -->
↑ [[GCP-MOC|GCP Mastery — Complete DevOps Notes]] · ← [[GCP-16-BigQuery|BigQuery]] · → [[GCP-18-Wrap-Up-Whats-Next|Wrap-Up — What's Next]]
<!-- /nav -->

> **Tags:** #gcp #cross-cloud #interview #trilateral #master-qa
> **Status:** The payoff section. Use this to demonstrate trilateral fluency in interviews.
> **Interview Relevance:** ⭐⭐⭐⭐⭐

## The Terminology Cheat Sheet (Three-Way)

| Category | AWS | Azure | GCP |
|---|---|---|---|
| **Account/Project boundary** | Account | Subscription | Project |
| **Org hierarchy** | Organization → OU → Account | Tenant → Management Group → Subscription → RG | Organization → Folder → Project |
| **Identity provider** | IAM + IAM Identity Center | Entra ID | Cloud Identity + IAM |
| **RBAC unit** | IAM Role / Policy | Azure RBAC Role Assignment | IAM Role Binding |
| **Machine identity** | IAM Role (assumed) | Managed Identity | Service Account |
| **VM** | EC2 | Virtual Machine | Compute Engine |
| **VM autoscaling group** | Auto Scaling Group | Virtual Machine Scale Set | Managed Instance Group |
| **Block storage** | EBS | Managed Disk | Persistent Disk |
| **Object storage** | S3 | Blob Storage | Cloud Storage |
| **File storage (NFS)** | EFS | Azure Files | Filestore |
| **Relational DB (managed)** | RDS / Aurora | Azure SQL / Azure DB for Postgres | Cloud SQL / AlloyDB |
| **NoSQL (key-value)** | DynamoDB | Cosmos DB (Table API) | Bigtable / Firestore (Datastore mode) |
| **NoSQL (document)** | DynamoDB (single-table) | Cosmos DB (Core / SQL API) | Firestore (Native mode) |
| **Globally consistent SQL** | Aurora Global (not truly consistent) | Cosmos DB (with Strong consistency, regional) | **Spanner (truly globally consistent)** |
| **Data warehouse** | Redshift | Synapse / Fabric | BigQuery |
| **Serverless functions** | Lambda | Azure Functions | Cloud Functions |
| **Serverless containers** | Fargate / App Runner | Container Apps | **Cloud Run** |
| **Managed Kubernetes** | EKS | AKS | GKE |
| **Container registry** | ECR | ACR | Artifact Registry |
| **Load balancer (L7)** | ALB | Application Gateway | Application Load Balancer (Global External) |
| **Load balancer (L4)** | NLB | Load Balancer (Standard) | Network Load Balancer (Passthrough) |
| **Global LB / CDN** | CloudFront + Global Accelerator | Front Door | **Global External LB + Cloud CDN (bundled)** |
| **CDN** | CloudFront | Azure CDN / Front Door | Cloud CDN |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |
| **Network (regional/global)** | VPC (regional) | VNet (regional) | **VPC (global)** |
| **Firewall (network)** | Security Group + NACL | NSG | VPC Firewall Rules |
| **API gateway** | API Gateway | API Management | API Gateway / Apigee |
| **Message queue** | SQS | Service Bus / Storage Queue | Pub/Sub (also does pub-sub) |
| **Pub-sub messaging** | SNS | Event Grid | Pub/Sub |
| **Streaming (Kafka-like)** | Kinesis / MSK | Event Hubs | Pub/Sub Lite |
| **Data pipeline (ETL)** | Glue / Step Functions | Data Factory / Synapse Pipelines | Dataflow / Cloud Composer |
| **IaC (native)** | CloudFormation / CDK | ARM / Bicep | Deployment Manager (deprecated) |
| **IaC (cross-cloud)** | Terraform | Terraform | Terraform |
| **CI/CD (native)** | CodePipeline + CodeBuild + CodeDeploy | Azure DevOps Pipelines | Cloud Build + Cloud Deploy |
| **Secrets manager** | Secrets Manager / Parameter Store | Key Vault | Secret Manager |
| **Monitoring (metrics/logs)** | CloudWatch | Azure Monitor / Log Analytics | Cloud Monitoring / Cloud Logging |
| **APM / tracing** | X-Ray | Application Insights | Cloud Trace |

## Cost Model Differences (One-Liner Each)

| Cloud | Discount Model | Compute Billing | Key Cost Wins |
|---|---|---|---|
| **AWS** | Reserved Instances / Savings Plans (1yr/3yr commit) | Per-second (60s min) | Spot Instances (up to 90% off), Savings Plans flexible across services |
| **Azure** | Reserved Instances (1yr/3yr) + Hybrid Benefit for Windows/SQL | Per-second | Hybrid Benefit slashes Windows/SQL licensing costs; Spot VMs |
| **GCP** | **Automatic Sustained Use Discounts** (up to 30% off, no commitment) + Committed Use Discounts (1yr/3yr) | Per-second (60s min) | **SUDs apply without you doing anything**; Custom Machine Types (pay for exact CPU/RAM you need); Spot VMs (up to 91% off) |

**Interview soundbite:** "GCP's cost model is the most forgiving — you get up to 30% off automatically just by running instances continuously, no upfront planning. AWS and Azure require you to actively commit to Reserved Instances or Savings Plans to get comparable discounts. Add Custom Machine Types on top and you're paying for exactly what you use."

---

## Master Q&A — Multi-Cloud Senior Interview Questions

### Q1: "You describe yourself as multi-cloud (AWS + Azure + GCP). Realistically, how do you decide which cloud for a workload?"

> "Honestly, the choice is usually made before I get involved — it's driven by enterprise agreements, existing ecosystem, or team skills. But when I actually have influence:
>
> **AWS by default** — broadest service catalog, most mature ecosystem, largest hiring pool. If nothing else drives the decision, AWS is the safe pick.
>
> **Azure when** — the org runs heavy Microsoft stack (M365, Windows Server, SQL Server, .NET, AD). Hybrid Benefit alone can save 40% on Windows/SQL licensing. Entra ID integration with corporate identity is unmatched.
>
> **GCP when** — data-heavy workloads (BigQuery is genuinely best-in-class), Kubernetes-first architecture (GKE Autopilot has no real equivalent), or teams that value operational simplicity (Cloud Run, sustained-use discounts, no cluster/DWU sizing exercises).
>
> For most orgs, the real answer is 'the cloud your team knows.' Cross-cloud migrations are expensive and rarely produce the savings people expect. My job is usually to make the chosen cloud work well, not to advocate for switching."

### Q2: "Design a 3-tier app on GCP for a moderate-scale SaaS workload."

> "Three tiers: web/API, application, data. Multi-region-ready but starting in `asia-south1`.
>
> **Networking:**
> - One custom-mode VPC (global, so it'll extend to additional regions later without redesign)
> - Regional subnet in asia-south1 with secondary ranges for GKE pods and services
> - Cloud NAT for outbound-only egress from private nodes
> - Firewall rules using service-account targeting (not tags) for anything security-relevant
>
> **Load balancing / edge:**
> - Global External Application Load Balancer with anycast IP
> - Managed SSL cert (Google auto-renews)
> - Cloud CDN toggled on for static content
> - Cloud Armor policies for WAF + DDoS
>
> **Compute (app tier):**
> - GKE cluster on Autopilot, Regular release channel
> - Main app deployed as Deployment with HPA on CPU + custom metric (request latency)
> - Workload Identity for pod-to-GCP authentication — no static secrets in K8s
> - Cloud SQL Auth Proxy sidecar for DB access
>
> **Data tier:**
> - Cloud SQL Postgres, Regional HA, private IP only (via Private Service Connect)
> - Read replicas for reporting queries
> - Daily automated backups + PITR
> - Firestore for user session data / real-time features (mobile clients get real-time listeners for free)
>
> **Async / eventing:**
> - Pub/Sub for inter-service events
> - Cloud Run services for lightweight event consumers (scale to zero, request-based billing)
>
> **Observability:**
> - Cloud Logging (aggregates GKE + Cloud Run + Cloud SQL logs automatically)
> - Cloud Monitoring dashboards + alerting
> - Cloud Trace for distributed tracing (auto-instrumented if using OpenTelemetry)
>
> **CI/CD:**
> - Cloud Build for CI (or GitHub Actions with Workload Identity Federation — no long-lived keys)
> - Artifact Registry for container images with vulnerability scanning
> - Cloud Deploy for progressive rollouts to GKE
>
> **Cost optimization from day one:**
> - Autopilot for main workloads — pay per pod, no idle capacity
> - Committed use discounts on the steady baseline of Cloud SQL and any Standard-mode node pools (if added later)
> - Spot node pool for batch/analytics workloads
> - Cloud CDN reduces egress costs meaningfully at scale
>
> **Compare this to my Azure 3-tier answer** (in [[Azure-MOC]]): fewer networking resources (no regional VNet duplication for multi-region readiness), simpler LB (one product, not App Gateway + Front Door), Autopilot has no direct Azure equivalent. Compare to AWS: no ALB Controller add-on needed for GKE Ingress, no separate NACL layer, one VPC instead of one per region."

### Q3: "When is GCP genuinely the right choice over AWS or Azure? Not marketing — real reasons."

> "Four scenarios where I'd actively recommend GCP:
>
> 1. **Data / analytics is the core workload.** BigQuery + Dataflow + Pub/Sub + Vertex AI is a coherent stack that competitors haven't matched. Redshift and Synapse work, but they require more operational effort and integrate less cleanly. If your product IS the data, GCP wins.
>
> 2. **Kubernetes-first architecture.** GKE — especially Autopilot — is measurably ahead of EKS and AKS in operational maturity, feature velocity, and cost efficiency for variable workloads. If 80%+ of your compute will be K8s, GCP is the strongest platform.
>
> 3. **Containers-and-scale-to-zero microservices.** Cloud Run has no true equivalent. App Runner and Container Apps come close but limit concurrency-per-instance, don't support arbitrary containers as flexibly, and have less mature routing/traffic-splitting. If your architecture is 'a mesh of stateless containerized services,' Cloud Run is a differentiator worth the platform choice.
>
> 4. **Globally-consistent transactional data.** Spanner is the only true answer to 'I need SQL with global strong consistency and horizontal scalability.' Aurora Global replicates but reads can be stale. Cosmos DB has strong consistency but only within a region. If you genuinely need this — global inventory, financial ledgers, gaming leaderboards — Spanner is the reason.
>
> **When NOT to pick GCP** (honest):
> - Windows/SQL Server-heavy workloads (Azure wins on licensing)
> - Deep AWS-service integration already (migrating won't pay off)
> - Team has no GCP skills and no time to train (rare skill in the Indian market — AWS is dominant)
> - Compliance requirements where AWS/Azure have more certifications in your specific region"

### Q4: "What's the biggest architectural mistake you see engineers make when moving from AWS or Azure to GCP?"

> "Not internalizing that VPCs are global. I've seen migrations where teams create one GCP project per AWS region — because that's what they'd do in AWS — and end up with N VPCs, N sets of firewall rules to keep in sync, VPC peering everywhere. It works, but it fights the platform.
>
> The GCP-native pattern: one project (or a small number, split by environment or business unit), one global VPC per project, subnets per region. Firewall rules defined once, targeted by service account. When you add a new region, you add a subnet — not a whole VPC.
>
> Second most common mistake: not using Workload Identity for pod-to-GCP auth in GKE. Teams port over patterns from EKS where they mounted IAM credentials into pods, or from AKS where they used aad-pod-identity. Workload Identity is simpler and more secure — just annotate the ServiceAccount — but requires understanding the SA-to-SA binding model.
>
> Third: over-using basic IAM roles (owner, editor, viewer). Teams from AWS are used to fine-grained IAM policies; on GCP they see 'Editor' as a convenient default and grant it broadly. Editor is nearly Owner — grants nearly all write access across nearly all services. Predefined roles or custom roles are the correct answer in production."

### Q5: "Compare cost optimization strategies across all three clouds."

> "AWS: commitment-based. Savings Plans and Reserved Instances are the primary discount lever — 40-72% off in exchange for 1-3 year spend commitments. Spot Instances for interruptible workloads (up to 90% off). Requires active planning and regular re-evaluation.
>
> Azure: commitment + licensing. Reserved Instances similar to AWS. Hybrid Benefit is the unique lever — if you have on-prem Windows Server or SQL Server licenses under Software Assurance, you can 'bring' them to Azure and eliminate the licensing portion of VM cost. For Windows-heavy shops, this alone can justify Azure over AWS. Spot VMs for interruptibles.
>
> GCP: passive + commitment. Sustained Use Discounts apply automatically — run a VM for a full month and you get up to 30% off, no commitment, no upfront work. Committed Use Discounts on top for another 20-30% if you can commit to CPU/RAM totals for 1-3 years (note: CUDs are commit-to-resource-totals, not commit-to-specific-machines like AWS RIs, which is more flexible). Custom Machine Types let you pay for exactly the CPU/RAM you need — no over-provisioning to the next tier. Spot VMs up to 91% off.
>
> Real-world rule of thumb: for equivalent workloads without any optimization, GCP tends to be 10-20% cheaper because of automatic SUDs. With aggressive optimization on all three, AWS can catch up via Savings Plans and Azure can beat both if you have Hybrid Benefit-eligible workloads. The 'cheapest cloud' answer depends entirely on your workload profile and licensing situation."
