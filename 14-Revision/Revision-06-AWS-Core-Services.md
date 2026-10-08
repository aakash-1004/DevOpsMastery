---
title: Revision — AWS Core Services
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 6
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-05-Kubernetes]]"
next: "[[Revision-07-Terraform]]"
related:
  - "[[AWS-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — AWS Core Services

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-05-Kubernetes|Revision — Kubernetes]] · → [[Revision-07-Terraform|Revision — Terraform (IaC)]]
<!-- /nav -->

## Concept

AWS is the dominant cloud provider. DevOps engineers primarily interact with: compute (EC2, ECS, Lambda), storage (S3, EBS, EFS), networking (VPC, Route53, CloudFront), and IAM.

**The mental model:** Every AWS resource lives in a region. Regions contain Availability Zones (physically separate data centers). High availability = spreading across AZs.

---

## IAM — Identity & Access Management

**Core concepts:**

- **User**: Human or machine identity with credentials
- **Role**: Assumable identity — for EC2 instances, Lambda functions, CI/CD tools
- **Policy**: JSON document defining permissions (Allow/Deny + Actions + Resources)
- **Group**: Collection of users sharing policies

**Least privilege principle:** Start with zero permissions. Grant only exactly what's needed.

```bash
# Check what your current identity can do
aws sts get-caller-identity              # Who am I?
aws iam list-attached-user-policies --user-name devops-admin  # What policies do I have?

# S3
aws s3 ls                               # List all buckets
aws s3 ls s3://mybucket/                # List bucket contents
aws s3 cp file.txt s3://mybucket/       # Upload file
aws s3 sync ./dist s3://mybucket/       # Sync local dir to S3
aws s3 rb s3://mybucket --force         # Delete bucket and all contents

# EC2
aws ec2 describe-instances              # List all instances
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name mykey \
  --security-group-ids sg-xxxxx \
  --subnet-id subnet-xxxxx              # Launch EC2 instance

aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress,Tags[?Key==`Name`].Value]' \
  --output table                        # Running instances with IPs and names
```

---

## VPC — Virtual Private Cloud

**Concepts:**

- **VPC**: Your private network in AWS (you define IP range, e.g., 10.0.0.0/16)
- **Subnet**: Subdivision of VPC. Public subnet = has route to Internet Gateway. Private subnet = no direct internet access.
- **Internet Gateway (IGW)**: Allows public subnets to communicate with the internet
- **NAT Gateway**: Allows private subnet instances to initiate outbound internet connections (for updates, etc.) without being reachable from outside
- **Security Group**: Stateful firewall at the instance level. Allow rules only.
- **NACL**: Stateless firewall at the subnet level. Allow and Deny rules.
- **Route Table**: Rules controlling where subnet traffic is directed

**3-tier architecture:**

```
Internet → IGW → Public Subnet (Load Balancer)
                      ↓
               Private Subnet (App Servers — EC2/ECS)
                      ↓
               Private Subnet (Database — RDS)
```

---

## S3 — Simple Storage Service

**S3 is object storage** — not a filesystem. Objects have keys (paths) and values (content). Max object size: 5TB.

**S3 storage classes (cost optimization):**

- **Standard**: Frequently accessed data. High cost, low latency.
- **Intelligent-Tiering**: Auto-moves between frequent/infrequent based on access patterns.
- **Standard-IA**: Infrequently accessed. Lower storage cost, retrieval fee.
- **Glacier**: Archives. Minutes to hours retrieval.
- **Glacier Deep Archive**: Coldest. 12-hour retrieval. Cheapest.

**Common DevOps uses:** Terraform remote state, CI/CD artifact storage, log archiving, static website hosting, Docker image layers (via ECR).

---

## Senior Interview Q&A

**Q: What's the difference between IAM Role and IAM User? When do you use each?**

> "IAM User is a permanent identity with long-lived credentials — access key + secret key. You use it for humans (developers, DevOps engineers) who log in to the console or use CLI locally.
>
> IAM Role is an identity that's assumed, not permanently held. It has no long-lived credentials — when assumed, it generates short-lived STS tokens (typically 1 hour). You use roles for:
>
> - EC2 instances (instance profile) — the instance assumes a role so your application code can call AWS APIs without storing keys on the instance
> - Lambda functions
> - ECS task execution
> - Cross-account access
> - GitHub Actions via OIDC (instead of storing AWS keys as GitHub Secrets)
>
> The golden rule: **never store AWS access keys on an EC2 instance or in code**. Always use IAM roles for machine-to-AWS authentication. If you see `aws_access_key_id` in a config file on a server, that's a security finding."

---

**Q: You need to give your EC2 app write access to S3. How do you do it?**

> "Create an IAM Role with an S3 policy. Attach the role to the EC2 instance as an instance profile.
>
> The policy would be: Action: `s3:PutObject`, `s3:GetObject`, Resource: `arn:aws:s3:::mybucket/*`. No `s3:*` — least privilege.
>
> The application uses the AWS SDK without any explicit credentials. The SDK automatically calls the EC2 instance metadata service (`169.254.169.254/latest/meta-data/iam/security-credentials/`) to get temporary credentials from the role. It refreshes them before expiry.
>
> Never add credentials to the application code or environment variables. The instance role handles it transparently."

---

**Q: Walk me through VPC design for a production application.**

> "I'd use a standard 3-tier architecture across two AZs minimum for HA.
>
> VPC CIDR: `10.0.0.0/16` — gives 65,536 addresses.
>
> Subnets across two AZs:
>
> - Public: `10.0.1.0/24` (AZ-a), `10.0.2.0/24` (AZ-b) — for ALB, NAT Gateway
> - Private App: `10.0.10.0/24` (AZ-a), `10.0.11.0/24` (AZ-b) — for EC2/ECS
> - Private DB: `10.0.20.0/24` (AZ-a), `10.0.21.0/24` (AZ-b) — for RDS
>
> Public subnets have a route to IGW. Private subnets route to NAT Gateway in the respective public subnet for outbound internet (package installs, API calls).
>
> Security groups: ALB accepts 443 from 0.0.0.0/0. App servers accept traffic only from ALB security group. DB accepts traffic only from App security group. Layered security — no direct public access to app servers or DB."

<!-- related -->
---
**Related:** [[AWS-MOC|AWS]]
<!-- /related -->
