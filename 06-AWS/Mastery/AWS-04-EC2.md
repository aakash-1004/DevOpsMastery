---
title: EC2 — Elastic Compute Cloud
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
order: 4
up: "[[AWS-Mastery-MOC]]"
prev: "[[AWS-03-IAM]]"
next: "[[AWS-05-EBS]]"
related:
  - "[[Lab-EC2-Deployment]]"
origin: DevOps-Mastery/06-AWS/AWS Mastery — Complete DevOps Notes.md
---

# EC2 — Elastic Compute Cloud

<!-- nav -->
↑ [[AWS-Mastery-MOC|AWS Mastery — Complete DevOps Notes]] · ← [[AWS-03-IAM|IAM — Identity & Access Management]] · → [[AWS-05-EBS|EBS — Elastic Block Store]]
<!-- /nav -->

## Concept

EC2 provides **resizable virtual servers** in the cloud, called **instances**. Instead of buying and managing physical servers, you rent VMs on demand.

Under the hood, EC2 runs on the Nitro hypervisor. Each instance you launch is a VM with dedicated CPU, RAM, storage, and network.

## Configuration Options

When launching an EC2 instance, you configure:

```
1. AMI (Amazon Machine Image)
   ├── The OS + pre-installed software
   ├── Ubuntu 22.04, Amazon Linux 2, Windows Server, custom AMI
   └── Determines what's installed at boot

2. Instance Type
   ├── The hardware capacity (CPU, memory, network)
   ├── t3.micro, m5.large, c5.xlarge, etc.
   └── Family + size (t = burstable, m = general, c = compute, r = memory)

3. Storage
   ├── Root EBS volume (default 8GB)
   ├── Additional EBS volumes if needed
   └── Type: gp3, gp2, io1, io2

4. Security Groups
   ├── Firewall rules — controls inbound/outbound traffic
   ├── Only "Allow" rules (no explicit deny)
   └── Stateful — return traffic auto-allowed

5. Key Pair
   ├── SSH access to Linux instances
   ├── RDP access for Windows instances
   └── Public key on instance, private key you download (.pem file)

6. Network Settings
   ├── VPC (Virtual Private Cloud) — your isolated network
   ├── Subnet (public or private)
   ├── Public IP assignment (yes/no)
   └── Elastic IP (static public IP, optional)

7. IAM Role
   ├── Attach role for AWS API access from the instance
   └── Never store credentials — always use role

8. User Data
   ├── Bootstrap script that runs at first boot
   ├── Installs software, configures the instance
   └── Runs as root
```

## Instance Type Families

```
t (t2, t3, t4g)  → Burstable — cheap, good for dev/test, low CPU with occasional bursts
m (m5, m6i)      → General purpose — balanced CPU/memory
c (c5, c6i)      → Compute optimized — CPU-heavy workloads (rendering, HPC)
r (r5, r6i)      → Memory optimized — databases, caching, analytics
i (i3, i4i)      → Storage optimized — NVMe SSD storage
p (p3, p4)       → GPU — machine learning, deep learning
g (g4, g5)       → GPU — graphics workloads
```

**Your context:** Your account requires `t3.micro` (not `t2.micro`) for free tier eligibility.

## User Data Script Example

```bash
#!/bin/bash
sudo yum update -y

# Install Apache web server
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd

# Create a simple HTML page
echo "<html><h1>Welcome to Apache on Amazon Linux!</h1></html>" > /var/www/html/index.html
```

**Real world context:** In production, user_data typically:

1. Pulls the latest application code from S3 or Git
2. Installs Docker
3. Starts your containerized app
4. Registers with a monitoring system
5. Reports success to CloudWatch

This is how Auto Scaling Groups launch new instances that are immediately ready to serve traffic — no manual setup.

## Security Groups (SG)

**What they are:** Virtual firewalls at the instance level.

**Rules:**

- Region-specific
- Only **Allow** rules (no explicit deny)
- All inbound blocked by default, all outbound allowed
- **Stateful** — if you allow inbound port 80, response traffic is automatically allowed outbound

**Define rules for:**

- Protocol (HTTP, HTTPS, SSH, custom TCP/UDP)
- Port number (80 for HTTP, 22 for SSH, 3306 for MySQL)
- Source (IP address, CIDR range, or another Security Group)

## Common Ports to Know

```
22    → SSH — Linux remote access
3389  → RDP — Windows remote access
80    → HTTP — unencrypted web
443   → HTTPS — encrypted web (SSL/TLS)
21    → FTP — file transfer (unsecured)
25    → SMTP — email sending
3306  → MySQL
5432  → PostgreSQL
1433  → MS SQL Server
27017 → MongoDB
6379  → Redis
```

**Interview trick question:** What port does SFTP use? Answer: **Port 22** — same as SSH (SFTP is SSH-based). Not port 21 (that's FTP).

## Practical: Creating an EC2 Instance

```bash
# Via AWS CLI
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-xxxxx \
  --subnet-id subnet-xxxxx \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]'
```

**Common EC2 commands:**

```bash
aws ec2 describe-instances                    # list all instances
aws ec2 start-instances --instance-ids i-xxx  # start stopped instance
aws ec2 stop-instances --instance-ids i-xxx   # stop running instance
aws ec2 terminate-instances --instance-ids i-xxx  # destroy permanently
```

## Stop vs Terminate

```
Stop
    ├── Instance shuts down but persists
    ├── EBS root volume preserved (data safe)
    ├── Public IP released (unless Elastic IP)
    ├── You still pay for EBS storage
    └── Can restart anytime

Terminate
    ├── Instance destroyed permanently
    ├── EBS root volume DELETED (unless DeleteOnTermination=false)
    ├── All data gone
    └── Stop paying immediately

Hibernate
    ├── RAM contents saved to EBS
    ├── Faster restart than stop
    └── Instance "remembers" its state
```

## Elastic IP

**Problem:** Public IP changes every time you stop/start an instance. **Solution:** Elastic IP = static public IP you own.

- Free while attached to a running instance
- Costs money if allocated but not used (AWS penalizes waste)
- Reassign to a different instance during failover

## Real World Production Setup

```
Web tier:
  m5.large × 3 instances (across 3 AZs)
  Behind ALB
  In Auto Scaling Group
  Public subnets

App tier:
  m5.xlarge × 5 instances
  Behind internal NLB
  Private subnets
  Reach out via NAT Gateway

Database tier:
  RDS Multi-AZ (not raw EC2)
  Private subnets
  Accessible only from app tier SG
```

## Memory Hints

- **Instance** = a VM (rented server)
- **AMI** = OS template (the "operating system in a box")
- **Instance Type** = hardware specs (t3.micro = 1 vCPU, 1GB RAM)
- **Security Group** = firewall for one instance (stateful)
- **Key Pair** = your house key (private key you keep, public key on the instance)
- **Elastic IP** = phone number that doesn't change (static)

## Interview Q&A — EC2

**Q: Walk me through what happens when you launch an EC2 instance.**

> "AWS receives the request via API. First, it validates permissions — your IAM policy must allow `ec2:RunInstances`. Then the scheduler picks a physical host in your specified AZ that has capacity for the instance type. The Nitro hypervisor allocates CPU, RAM, and network resources.
>
> The AMI is copied to the instance's root EBS volume. Instance boots up, user_data script runs as root. Security Group rules are applied — inbound traffic filtered per your rules. If public IP is enabled, it's assigned. Instance transitions from `pending` to `running` state.
>
> Total time — usually 30-60 seconds for standard AMIs, longer for large custom AMIs. This is why AMIs matter — a well-built AMI with everything pre-installed launches in seconds vs. minutes for a bare AMI that needs user_data to install everything."

**Q: Stop vs Terminate an EC2 instance — what happens to data?**

> "Stop preserves everything on EBS volumes. The instance shuts down like a laptop hibernating — you can start it again later, and all data is there. You still pay for EBS storage even when stopped. Public IP changes on restart unless you have an Elastic IP attached.
>
> Terminate destroys the instance permanently. The root EBS volume is deleted by default — data gone. You can override this by setting DeleteOnTermination to false on the volume, but this is unusual.
>
> In production, we terminate instances all the time — Auto Scaling Groups terminate old instances during rolling updates, on-demand instances get terminated when demand drops. The rule is: never store data on an EC2 root volume that you can't afford to lose. Persistent data goes to EBS with DeleteOnTermination=false, or to S3, or to RDS."

**Q: How does user_data help in Auto Scaling scenarios?**

> "User_data is the bootstrap script that runs when an instance first launches. In Auto Scaling, this is critical because new instances need to be production-ready immediately — no human intervention.
>
> Typical user_data flow: update OS packages, install Docker, pull the latest application image from ECR, start the container with proper environment variables, and register with monitoring. When ASG scales up during traffic spike, new instances follow this exact flow and start serving traffic in 1-2 minutes.
>
> The alternative is baking everything into a custom AMI — faster launch (30 seconds vs 2 minutes) but slower iteration since you rebuild the AMI for every app change. The pattern I recommend: bake OS + Docker + monitoring agents into the AMI, use user_data only to pull latest code. Best of both worlds."

<!-- related -->
---
**Related:** [[Lab-EC2-Deployment|Lab — Deploying Taskmanager to AWS EC2]]
<!-- /related -->
