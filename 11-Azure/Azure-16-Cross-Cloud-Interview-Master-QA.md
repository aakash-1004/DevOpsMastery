---
title: "Cross-Cloud Interview Master Q&A"
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
order: 16
up: "[[Azure-MOC]]"
prev: "[[Azure-15-Networking-VNets-Subnets-NSGs]]"
next: "[[Azure-17-Session-Wrap-up]]"
origin: DevOps-Mastery/11-Azure/Azure Mastery.md
---

# Cross-Cloud Interview Master Q&A

<!-- nav -->
↑ [[Azure-MOC|Azure Mastery — Complete DevOps Notes]] · ← [[Azure-15-Networking-VNets-Subnets-NSGs|Networking — VNets, Subnets, NSGs]] · → [[Azure-17-Session-Wrap-up|Session Wrap-up]]
<!-- /nav -->

## "You're an AWS shop being asked to also support Azure — what changes in your mental model?"

> "Three big mental shifts, then a lot of pattern matching.
>
> Shift one: the resource hierarchy. AWS is Account → Resources with tags providing organization. Azure inserts Subscription → Resource Group → Resource. The Resource Group is the load-bearing new concept — every resource belongs to exactly one RG, and RGs enable atomic cleanup, natural RBAC scoping, and clean environment separation. Once you internalize RGs, a lot of Azure operations click.
>
> Shift two: identity model. AWS IAM is one service handling both identity and resource authorization. Azure splits: Entra ID for identity, RBAC for authorization on resources. Entra ID is a proper enterprise identity provider — it also does Microsoft 365, third-party SaaS SSO. Bigger scope than AWS IAM. RBAC uses pre-built roles (Contributor, Reader, service-specific) assigned to scopes (subscription/RG/resource) that inherit downward. Simpler than writing JSON policies for most cases.
>
> Shift three: managed services are more numerous and more integrated. AWS bundles many capabilities into a few services (ALB does path routing, host routing, WAF integration). Azure splits into more services (Load Balancer for L4, Application Gateway for regional L7, Front Door for global L7, Traffic Manager for DNS routing). More services to learn, but each is more focused. Once you know which one to reach for, they compose well.
>
> Then there's the pattern matching — most services have direct AWS equivalents. EC2 = VM, S3 = Blob Storage, RDS = Azure SQL, Lambda = Functions, EKS = AKS, CloudFront = Front Door, Route 53 = Azure DNS + Traffic Manager. Once you've mapped the vocabulary, day-to-day work is very similar. The Kubernetes layer is identical.
>
> One free win: AKS control plane is free. EKS is $73/month per cluster. This changes multi-cluster architecture economics."

---

## AWS ↔ Azure Terminology Cheat Sheet

| AWS | Azure | Notes |
|---|---|---|
| Account | Subscription | Billing + quota boundary |
| Organization | Entra ID Tenant + Management Groups | Multi-account governance |
| (implicit / tags) | **Resource Group** | Mandatory logical container in Azure |
| Region | Region | Same concept |
| Availability Zone | Availability Zone | Same concept, not universal in Azure |
| IAM User | Entra ID User | |
| IAM Group | Entra ID Group | |
| IAM Role | Service Principal / Managed Identity | Managed Identity is preferred for Azure resources |
| IAM Policy | RBAC Role Assignment | Azure uses pre-built roles + scopes mostly |
| Instance Profile | System-assigned Managed Identity | |
| EC2 | Virtual Machine | |
| EBS | Managed Disk | |
| AMI | VM Image / Shared Image Gallery | |
| Auto Scaling Group | VMSS (VM Scale Set) | VMSS bundles launch template + ASG |
| Launch Template | (embedded in VMSS model) | |
| S3 | Blob Storage | Note: Storage Account wraps containers |
| S3 Bucket | Container (in Storage Account) | |
| Glacier | Archive access tier | |
| EFS | Azure Files | SMB shares |
| RDS | Azure SQL / Database for MySQL / PostgreSQL | |
| Aurora | Azure SQL Business Critical / Hyperscale | Hyperscale is closest |
| DynamoDB | Cosmos DB (NoSQL API) | Cosmos is more feature-rich |
| ElastiCache | Azure Cache for Redis | |
| Redshift | Azure Synapse Analytics | |
| Lambda | Azure Functions | |
| ECS | Azure Container Apps / Container Instances | Container Apps for microservices, ACI for one-off |
| EKS | AKS | Control plane FREE on AKS |
| Fargate | Container Apps / Container Instances / AKS Virtual Nodes | |
| API Gateway | API Management | Azure APIM is more feature-rich but more expensive |
| Route 53 | Azure DNS + Traffic Manager | Split into two services |
| ALB | Application Gateway | |
| NLB | Load Balancer (Standard) | |
| CloudFront | Front Door (or Azure CDN — sunsetting) | Front Door is more capable |
| Global Accelerator | Front Door | (bundled) |
| WAF | Front Door WAF / Application Gateway WAF | |
| VPC | VNet | Regional in both |
| Subnet | Subnet | |
| Security Group | NSG (partly) | NSG allows AND denies |
| NACL | NSG (also this — NSG combines both) | |
| Internet Gateway | (implicit) | Subnets get internet via public IP or NAT |
| NAT Gateway | NAT Gateway | Same |
| VPC Peering | VNet Peering | |
| VPC Endpoint / PrivateLink | Private Endpoint / Private Link | |
| Systems Manager Session Manager | Azure Bastion | Managed jump host |
| CloudFormation | ARM / Bicep | Bicep is the modern DSL |
| CDK | Bicep (roughly) | Both compile to underlying JSON |
| Systems Manager Parameter Store | Azure App Configuration | |
| Secrets Manager | Azure Key Vault | Key Vault also stores certs and keys |
| KMS | Azure Key Vault (keys section) | |
| CloudWatch | Azure Monitor + Log Analytics + App Insights | Three products split by concern |
| CloudTrail | Azure Activity Log + Diagnostic Logs | |
| GuardDuty | Microsoft Defender for Cloud | |
| Config | Azure Policy + Azure Resource Graph | |
| CloudTrail Insights | Microsoft Sentinel | SIEM |
| SNS | Azure Service Bus / Event Grid | Event Grid for events, Service Bus for messages |
| SQS | Service Bus Queues / Storage Queues | |
| Kinesis | Event Hubs | |
| Step Functions | Logic Apps / Durable Functions | Logic Apps for low-code, Durable Functions for code-first |
| CodePipeline | Azure Pipelines / GitHub Actions | |
| CodeBuild | Azure Pipelines / GitHub Actions | |
| ECR | Azure Container Registry (ACR) | |
| CodeCommit | Azure Repos | (Both losing to GitHub) |
| CloudFormation StackSets | Deployment Stacks | |

## Cost Model Differences Worth Knowing

**1. Reserved Capacity is Cheaper in Azure (for equivalent commitment).** Both clouds offer 1-year and 3-year reserved pricing. Azure's savings are often slightly higher for equivalent VM SKUs, especially with Azure Hybrid Benefit (reuse existing on-prem Windows Server / SQL Server licenses).

**2. Azure Free Services (that AWS Charges For):**
- AKS control plane (EKS is $73/mo)
- Front Door base tier includes WAF (AWS WAF is separate charge)
- Azure DevOps free tier is generous for CI/CD (compared to CodeBuild minutes)

**3. AWS Wins on Some Specifics:**
- S3 Standard is cheaper per GB than Blob Storage Hot (marginal but real)
- Lambda free tier is more generous than Functions Consumption free tier
- EBS gp3 is cheaper baseline than Premium SSD

**4. Data Egress — Similar Structure, Both Expensive**
Both charge for outbound data to the internet. Both are free for inbound. Both have "free" data movement within region. Charges kick in for cross-region and cross-cloud data. Same order of magnitude, ~$0.08-0.09/GB.

**5. Watch for Storage Account Redundancy Cost Multipliers**
LRS is 1x. ZRS is ~1.25x. GRS is ~2x. GZRS is ~2.5x. Choose deliberately — a bank might justify GZRS, a dev environment definitely doesn't need it. Same principle in S3 (Standard vs Standard-IA vs Cross-Region Replication).

**6. Reserved Instances vs Savings Plans**
Azure has Reserved Instances (specific VM series in a region for 1/3 years) and Savings Plans (commit to hourly compute spend). AWS has similar. Savings Plans are more flexible; RIs are stricter but cheaper. Same trade-off.

---

## Design Question: 3-Tier Web Application on Azure

**Q: Design a production-grade 3-tier web app on Azure. Highly available, secure, cost-conscious.**

> "Full architecture, walking through region choice first, then each tier.
>
> Region: East US primary, West US secondary for DR. Both have Availability Zone support — critical for HA design. Same region for prod resources unless there's a specific latency need.
>
> Resource groups: one per environment per lifecycle grouping.
> - `rg-prod-network-eastus` — VNet, NSGs, DNS zones (shared, rarely changes)
> - `rg-prod-web-eastus` — App Gateway, Front Door, web-tier resources
> - `rg-prod-app-eastus` — AKS cluster, container images, application secrets
> - `rg-prod-data-eastus` — Azure SQL, Cosmos DB, Blob Storage, Key Vault
>
> Networking layer:
> - VNet: 10.0.0.0/16
> - Subnets: web /24, app /24, db /24, bastion /27, appgw /24
> - NSGs enforce tier isolation with NSG-as-source rules
> - NAT Gateway for outbound from private tiers
> - Private DNS zones for internal service discovery
> - Azure Bastion for admin access
>
> Web tier:
> - Front Door (global L7) as public entry point — DNS points here
> - WAF policy on Front Door: OWASP + custom rules
> - Origin: Application Gateway in web subnet
> - Application Gateway → AKS backend pool via AGIC (Application Gateway Ingress Controller)
> - Managed certificates on both Front Door and App Gateway
>
> App tier:
> - AKS cluster, Standard tier with uptime SLA, Kubernetes 1.29
> - Node pools: system (3 nodes, D2s_v5), general (autoscale 3-15, D4s_v5), spot (autoscale 0-30 for batch)
> - All node pools spread across 3 AZs
> - Entra ID authentication + Azure RBAC for Kubernetes
> - Workload Identity for pod-level auth to Azure resources
> - Container Insights for observability
> - ArgoCD for GitOps deployment
> - Cert-manager for TLS certificates
>
> Data tier:
> - Azure SQL Database Business Critical (Gen5, 4 vCore, zone-redundant) for primary transactional data
> - Read replica for reporting workloads
> - Backup retention: 35 days short-term + 12 months long-term
> - Cosmos DB (NoSQL API) for session state and cart data — global replication to West US, session consistency
> - Blob Storage account (GZRS) for user uploads, lifecycle policy transitions to Cool after 30 days
> - All data services accessed via Private Endpoints — no public exposure
>
> Security layer:
> - Key Vault for secrets, certificates, encryption keys
> - Managed Identity everywhere — no service accounts with passwords
> - Azure Policy enforces compliance (no public storage, no public DBs, tag requirements)
> - Microsoft Defender for Cloud for threat detection
> - Sentinel for SIEM and security analytics
> - Enable diagnostic logs on everything, sent to central Log Analytics workspace
>
> DR strategy:
> - RTO 30 min, RPO 5 min for critical data
> - West US region has warm-standby AKS cluster (small, autoscales up on failover)
> - Azure SQL geo-replication to West US
> - Cosmos DB is already global (multi-region writes)
> - Blob Storage GZRS replicates automatically to paired region
> - Traffic Manager or Front Door priority routing for automatic failover
> - Terraform manifests + Bicep in Git — full infrastructure recreatable in 30 min
>
> Observability:
> - Azure Monitor as the umbrella
> - Log Analytics workspace as central log store
> - Application Insights for app-level telemetry, traces, metrics
> - Container Insights for AKS-specific
> - Alert rules → PagerDuty via webhook
> - Grafana on top of Log Analytics for custom dashboards
>
> CI/CD:
> - GitHub Actions for CI (build, test, push to ACR)
> - ArgoCD in AKS for CD (pull-based GitOps)
> - Separate service principals per pipeline with least-privilege RBAC
> - OIDC federation to Entra ID — no long-lived secrets
>
> Cost optimization notes:
> - VMs behind AKS as Reserved Instances (1-year commit) — 40% off baseline
> - Spot node pool absorbs batch and elastic workloads — 80% off
> - Azure SQL vCore purchasing model with Azure Hybrid Benefit if applicable
> - Storage lifecycle policies save on old data
> - Regularly review Azure Advisor cost recommendations
>
> This architecture achieves ~99.95% availability, defense-in-depth security, sub-100ms global latency via Front Door, automatic failover for regional outages, and clean cost profile.
>
> Direct AWS comparison: this is the same architecture as CloudFront + ALB + EKS + RDS Multi-AZ + DynamoDB Global Tables + S3 CRR. Different service names, identical shape. That transferability is exactly why cloud-agnostic architecture skills matter."
