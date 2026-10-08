---
title: AMI — Amazon Machine Image
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
order: 6
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-05-EBS]]"
next: "[[AWS-07-ELB]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# AMI — Amazon Machine Image

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-05-EBS|EBS — Elastic Block Store]] · → [[AWS-07-ELB|ELB — Elastic Load Balancer]]
<!-- /nav -->

## Concept

An **AMI (Amazon Machine Image)** is a pre-configured template that provides everything needed to launch an EC2 instance:

- Operating system (Ubuntu, Amazon Linux, Windows)
- Pre-installed applications (Apache, Docker, etc.)
- Configuration files and settings

Think of an AMI as a snapshot of a fully configured machine that you can duplicate.

## Types of AMIs

```
Public AMIs
    ├── Available to all AWS users
    ├── Provided by AWS or community
    ├── Examples: Amazon Linux 2, Ubuntu 22.04, Windows Server 2022
    └── Use for: standard OS deployments

Private AMIs
    ├── Created by you, only in your account
    ├── Can be shared with specific AWS accounts
    ├── Contains your custom configuration
    └── Use for: production golden images

Paid AMIs / Marketplace AMIs
    ├── Third-party provided via AWS Marketplace
    ├── Include software licenses (RHEL, LAMP stack, Bitnami apps)
    ├── Extra hourly cost for the software
    └── Use for: enterprise software, rapid deployment
```

## Real Production Use Case — LAMP Stack Paid AMI

**Scenario:** You need to deploy a PHP web app quickly.

Using a paid LAMP AMI from AWS Marketplace:

- **Rapid Deployment:** Apache, MySQL, PHP pre-configured — skip installation
- **Scalability:** Launch multiple instances behind ALB in minutes
- **Cost:** Pay for the AWS instance + software license (usually small hourly fee)

## Creating a Custom AMI

**Workflow:**

1. Launch an EC2 instance from a base AMI (e.g., Amazon Linux 2)
2. Install your applications, configure everything
3. Run cleanup script (remove SSH keys, credentials, temp files)
4. Create an AMI from the instance
5. Launch new instances from your custom AMI — they come up fully configured

**Cleanup Script Before Creating AMI (CRITICAL):**

```bash
#!/bin/bash

# Remove SSH keys
rm -rf ~/.ssh/authorized_keys

# Clear user credentials and history
rm -rf ~/.aws/credentials
rm -rf ~/.git-credentials
rm -rf ~/.bash_history

# Clean system logs and temp files
rm -rf /var/log/*
rm -rf /tmp/*
rm -rf /var/tmp/*

# Remove temporary user accounts
deluser tempuser --remove-home

# Lock root account
passwd -l root

# Reset app-specific configs if needed
# rm -rf /etc/nginx/nginx.conf
```

**Why cleanup matters:** Without it, everyone who launches from your AMI gets your credentials, your bash history (which might contain passwords), your SSH keys. **This is a major security incident** — happens more often than you'd think in real companies.

## Launch Template vs Custom AMI

```
Launch Template
    ├── AWS resource that stores instance configuration
    ├── Doesn't include the OS/apps — points to an AMI
    ├── Reusable — Auto Scaling uses it to launch instances
    ├── Versioned — you can create v1, v2, v3 and switch between them
    └── Use for: standardizing how instances are launched

Custom AMI
    ├── The actual OS image with apps pre-installed
    ├── Used BY Launch Template
    └── Use for: bundling application code and dependencies

Common pattern:
    Custom AMI (contains OS + Docker + monitoring)
        ↓ referenced by
    Launch Template (instance type, security group, IAM role, user_data)
        ↓ used by
    Auto Scaling Group (launches instances based on template)
```

## EC2 Image Builder

**What it is:** AWS service to automate creation, testing, and deployment of AMIs.

**Features:**

- Runs at scheduled intervals (daily, weekly, monthly)
- Can automatically apply security patches to base AMIs
- Tests the AMI before publishing
- **Free service** — you only pay for the EC2 instances used during the build

**Real world context:** Enterprises use Image Builder to maintain a "golden AMI" — always patched, always up to date. Every month, Image Builder auto-generates a new AMI with latest security patches, tests it, and marks it as the latest. Auto Scaling Groups auto-adopt the new AMI. **Zero manual patching**.

## Immutable Infrastructure Pattern

**Traditional (mutable):**

```
Launch instance → SSH in → install updates → change configs → over time it drifts
```

**Immutable (with AMIs):**

```
Build new AMI with all changes → launch new instances from it → terminate old instances
Never modify running instances — always replace them
```

**Benefits:**

- Consistent state across all instances
- Rollback = launch instances from old AMI
- No configuration drift
- Faster incident response

## Memory Hints

- **AMI** = "OS snapshot with everything baked in"
- **Launch Template** = "recipe for how to bake a cake" (references the AMI)
- **AMI** = the cake, **Launch Template** = the recipe
- **Image Builder** = automated bakery (produces AMIs on schedule)

## Interview Q&A — AMI

**Q: What's the difference between a custom AMI and a Launch Template?**

> "They serve different purposes and work together.
>
> Custom AMI is the actual OS image — contains the operating system, all your installed applications, and configuration. Think of it as a snapshot of a fully configured server.
>
> Launch Template is a reusable configuration for how to launch an instance — specifies the AMI ID, instance type, security group, IAM role, user_data, and tags. It doesn't contain the OS itself, just points to which AMI to use.
>
> Common pattern in production: build a custom AMI with Docker + monitoring agents + security tools baked in. Then create a Launch Template that references that AMI and specifies instance type m5.large, sec group X, IAM role Y. Auto Scaling Group uses the Launch Template to spawn identical instances on demand.
>
> Update flow: rebuild AMI monthly with latest patches, update Launch Template to point to new AMI, ASG rolls new instances gradually."

**Q: How do you achieve immutable infrastructure with AMIs?**

> "Immutable infrastructure means you never modify running servers — you replace them.
>
> Build phase: create a custom AMI with your app code, dependencies, and configuration baked in. Version this AMI (e.g., myapp-v1.2.3-2026-07-16). Test it in a staging environment.
>
> Deploy phase: update the Launch Template to reference the new AMI. Trigger a rolling update in the Auto Scaling Group — ASG launches new instances with the new AMI, terminates old ones. Zero downtime if you have multiple instances behind a load balancer.
>
> Benefits: consistent state (every instance is identical), easy rollback (point Launch Template back to old AMI, re-roll), no config drift (nobody SSHs in to change things), fast incident response (bad deploy = rollback in minutes).
>
> This is why tools like Packer exist — automate AMI building as part of CI/CD. Your pipeline: code push → Packer builds AMI → tests → updates Launch Template → ASG rolling update."
