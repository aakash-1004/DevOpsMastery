---
title: "Interview Master Q&A"
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
order: 19
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-18-Amplify]]"
next: "[[AWS-20-Session-Wrap-up]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# Interview Master Q&A

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-18-Amplify|Amplify — Fullstack Platform]] · → [[AWS-20-Session-Wrap-up|Session Wrap-up]]
<!-- /nav -->

## Foundation Questions

**Q: What is a Region and Availability Zone in AWS? Why do they matter?**

> "A Region is a physical geographic location (like Mumbai, N. Virginia, London). Each Region contains multiple Availability Zones — physically isolated data centers with independent power, cooling, and networking. AZs in the same region are connected via low-latency links.
>
> They matter for high availability. Deploying across multiple AZs protects against single-datacenter failures. If AZ-a in Mumbai has a power outage, your workloads in AZ-b and AZ-c continue serving traffic. This is the foundation of cloud reliability — always design for multi-AZ.
>
> For truly critical apps, deploy multi-region — protects against entire region outages. But most workloads are fine with multi-AZ, which handles the far more common AZ-level failures. Multi-region adds significant complexity and cost, so justify it with actual availability requirements."

**Q: What is the difference between horizontal and vertical scaling?**

> "Vertical scaling — scaling up. Add more power (CPU, RAM, disk) to an existing server. Example: upgrade EC2 from t3.medium to m5.xlarge. Simple but has limits — you can only get so big, and it usually requires downtime.
>
> Horizontal scaling — scaling out. Add more servers behind a load balancer. Example: EC2 count from 3 to 30 instances behind ALB. No hardware limits, no downtime, can scale essentially infinitely.
>
> Cloud makes horizontal scaling trivial — Auto Scaling Groups handle it automatically. That's why cloud native apps are designed to be stateless — any instance can serve any request, so horizontal scaling just works."

## IAM Deep Dive

**Q: Walk me through best practices for an IAM setup in production.**

> "First, protect the root account — enable MFA, never use it for daily operations, don't create access keys for it. Store the root password in a secrets manager. Use it only for account-level actions like billing.
>
> Second, principle of least privilege everywhere. Never grant AdministratorAccess unless truly needed. Start with zero permissions, add specific policies as needed.
>
> Third, use IAM roles for AWS services, never access keys. EC2 instances use instance profiles. Lambda functions use execution roles. GitHub Actions uses OIDC to assume roles temporarily. No long-lived credentials on servers.
>
> Fourth, use Groups for humans. Create a Developers group with dev-appropriate policies. Add developers to the group. Removes user offboarding — remove from group, they lose access. Cleaner than managing per-user policies.
>
> Fifth, enable MFA for all human users, especially admins. Consider hardware tokens (YubiKey) for admin accounts.
>
> Sixth, audit regularly. Use IAM Access Analyzer to find overly permissive policies. AWS Config to detect drift. CloudTrail to audit all API calls.
>
> Seventh, credential rotation — rotate access keys quarterly for programmatic access that must use keys. Better yet, replace with roles where possible."

## Networking

**Q: Explain a production VPC design for a 3-tier web application.**

> "Standard reference architecture, multi-AZ for high availability.
>
> VPC CIDR: 10.0.0.0/16 — 65,536 addresses, plenty of room.
>
> Six subnets across two AZs (minimum, three AZs for higher availability): Public subnets: 10.0.1.0/24 (AZ-a), 10.0.2.0/24 (AZ-b) — for ALB, NAT Gateway Private App subnets: 10.0.10.0/24 (AZ-a), 10.0.11.0/24 (AZ-b) — for EC2/ECS Private DB subnets: 10.0.20.0/24 (AZ-a), 10.0.21.0/24 (AZ-b) — for RDS
>
> Public subnets have route to Internet Gateway. Private subnets route to NAT Gateway (in respective public subnet) for outbound internet — needed for package updates, external API calls.
>
> Security groups layered: ALB SG allows 443 from 0.0.0.0/0. App SG allows traffic only from ALB SG. DB SG allows traffic only from App SG. Traffic path enforced by security groups — no direct public access to app or DB tier.
>
> NACLs at subnet level for additional defense in depth if compliance requires. Otherwise, security groups alone are sufficient for most workloads.
>
> This design gives you: high availability across 2 AZs, defense in depth with layered security groups, protected database tier, controlled outbound internet access via NAT Gateway, and clean separation of concerns."

## Storage

**Q: Compare EBS, S3, and EFS. When would you use each?**

> "Different storage types for different use cases.
>
> EBS: block storage, attached to one EC2 instance at a time. Persistent, high performance. Use for OS root volumes, databases, application data on EC2. Not shareable — each EBS volume is exclusively attached to one instance (with rare exceptions).
>
> S3: object storage, accessed via HTTP API. Infinitely scalable, 11 nines durability, region-wide access. Use for: application file storage, backups, static websites, data lakes, logs. Not a filesystem — no mount, no directories in the traditional sense.
>
> EFS: file storage, NFS-based shared filesystem. Multiple EC2 instances can mount it simultaneously. Use for: shared configuration, ML training data, WordPress on multiple instances, application logs that multiple services write to.
>
> Cost hierarchy: S3 cheapest (especially Glacier), EBS middle, EFS most expensive.
>
> Modern architecture rarely uses EFS — instead use S3 for shared storage via API. EFS is legacy pattern for lift-and-shift of NFS-based apps."

## Cost Optimization

**Q: An engineering VP asks how to reduce AWS costs. What's your approach?**

> "Systematic approach starting from highest-impact levers.
>
> First, right-size compute. AWS Compute Optimizer analyzes CloudWatch data and recommends smaller instance types. Typical savings 20-40% just by matching instance size to actual usage. Auto Scaling Groups handle variable load — don't over-provision baseline.
>
> Second, use Reserved Instances or Savings Plans for predictable workloads. If EC2 instances run 24/7, RI gives 40-60% savings for 1-year commitment. Savings Plans are even more flexible — commit to spend rather than specific instances.
>
> Third, Spot Instances for fault-tolerant workloads — up to 90% discount. Batch jobs, ML training, dev/test environments. Auto Scaling supports mixing On-Demand and Spot.
>
> Fourth, storage optimization. S3 lifecycle policies move old data to Glacier — massive savings. Delete unattached EBS volumes (they still cost money). Right-size EBS — over-provisioned volumes are common.
>
> Fifth, delete unused resources. Unused Elastic IPs cost money. Idle load balancers. Old snapshots. AWS Cost Explorer identifies these.
>
> Sixth, use Serverless where appropriate. Lambda for sporadic workloads instead of always-on EC2. Fargate for containers with variable load. DynamoDB on-demand for unpredictable traffic. Pay only for actual use.
>
> Seventh, network egress costs. Data transfer OUT to internet is expensive ($0.09/GB). CloudFront caching reduces origin egress. VPC endpoints eliminate charges for S3/DynamoDB traffic within region.
>
> Eighth, cross-account visibility. AWS Cost Explorer, tagging strategy, budgets and alerts. Nobody spends what they don't measure.
>
> Real result: I've seen teams reduce AWS costs 30-50% with 3-6 months of systematic optimization without any performance impact."

## Availability & Reliability

**Q: How would you design a highly available web application on AWS?**

> "Reference architecture achieving 99.99% availability.
>
> DNS layer: Route 53 with health checks and failover routing. Optionally multi-region with latency-based routing for global users.
>
> Edge layer: CloudFront CDN in front of everything. Caches static content globally, provides DDoS protection via AWS Shield, HTTPS via ACM certificate.
>
> Load balancer layer: Application Load Balancer, multi-AZ, spread across 3 AZs. Cross-zone load balancing enabled.
>
> Compute layer: Auto Scaling Group with EC2 instances across 3 AZs. Min 3 instances, max scales with demand. Instances behind ALB. Rolling deployments via Launch Template versions. Or ECS/EKS for container workloads.
>
> Database layer: RDS Multi-AZ for production database. Automatic failover in ~60 seconds. Read replicas for scale. Cross-region read replica for DR.
>
> Storage: S3 for object storage with cross-region replication for critical data. EBS with snapshots for EC2 data volumes.
>
> Monitoring: CloudWatch metrics and alarms. SNS notifications to on-call. X-Ray for distributed tracing.
>
> DR: Route 53 failover to a warm standby in another region. Terraform templates to spin up full replica if needed. RTO under 15 minutes, RPO under 5 minutes for critical data.
>
> This architecture handles single instance failures, AZ failures, and even entire region failures. 99.99% is achievable, and if you eliminate every SPOF and add multi-region active-active, 99.999% becomes possible."

## Disaster Recovery

**Q: Explain the four DR strategies and their trade-offs.**

> "Four strategies with progressively lower RTO/RPO and higher cost.
>
> Backup and Restore: cheapest, slowest. Regular backups to S3, restore in another region during disaster. RTO hours-days, RPO hours. Good for non-critical workloads.
>
> Pilot Light: minimal infrastructure always running in DR region. Core components like databases replicating. During disaster, scale up other components. RTO 10-30 min, RPO minutes. Moderate cost.
>
> Warm Standby: scaled-down copy of production always running in DR region. Ready to scale up quickly. RTO 5-15 min, RPO seconds-minutes. Higher cost.
>
> Multi-Region Active-Active: full production in multiple regions, all serving traffic. Route 53 latency-based routing. Instant failover. RTO seconds, RPO near-zero. Highest cost — essentially double infrastructure.
>
> How to choose: match strategy to business needs. E-commerce during holiday season → active-active worth the cost. Internal HR system → backup and restore is fine.
>
> Most companies use warm standby for critical services and backup/restore for non-critical. Multi-region active-active is reserved for the most demanding workloads."

## Security

**Q: How do you secure a web application on AWS?**

> "Defense in depth across multiple layers.
>
> Layer 1: Network isolation. VPC with private subnets for app and DB tiers. Public subnets only for load balancers and NAT Gateway. Security groups enforce least-privilege network access — DB SG only accepts traffic from App SG.
>
> Layer 2: Edge security. AWS Shield Standard (free with CloudFront) blocks common DDoS. AWS WAF filters malicious HTTP traffic — SQL injection, XSS, bot traffic. Rate limiting on API endpoints.
>
> Layer 3: Identity and access. IAM roles for service-to-service auth — no hardcoded credentials. Cognito for user authentication. MFA required for admin accounts.
>
> Layer 4: Encryption. HTTPS everywhere with ACM certificates. EBS volumes encrypted at rest with KMS. S3 buckets encrypted. RDS encrypted. Secrets in Secrets Manager, not env vars.
>
> Layer 5: Monitoring and audit. CloudTrail logs all API calls. GuardDuty detects threats via ML. AWS Config tracks configuration changes. Access logs from ALB and CloudFront to S3 for forensics.
>
> Layer 6: Vulnerability management. ECR image scanning for containers. Amazon Inspector for EC2 vulnerabilities. Regular patching of AMIs.
>
> Layer 7: Data protection. VPC endpoints for S3/DynamoDB — traffic never leaves AWS. S3 bucket policies block public access by default.
>
> Ongoing: security audits quarterly. Incident response plan tested. Team trained on security. Compliance as needed (SOC 2, PCI DSS, HIPAA)."
