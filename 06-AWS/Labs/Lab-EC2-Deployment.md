---
title: Lab — Deploying Taskmanager to AWS EC2
aliases:
  - EC2-Deployment-Lab
type: lab
module: aws
status: ✅ Completed
tags:
  - aws
  - ec2
  - docker
  - lab
  - day4
  - golden-thread
repo: "https://github.com/aakash-1004/taskmanager"
instance: "`devops-lab` — ap-south-1 (Mumbai)"
up: "[[AWS-MOC]]"
prev: "[[6.2-AWS-Deployment-EC2-ECS]]"
related:
  - "[[6.1-AWS-Core-Services]]"
  - "[[AWS-04-EC2]]"
  - "[[Lab-Taskmanager-Dockerization]]"
  - "[[7.2-GitHub-Actions-Pipeline]]"
origin: DevOps-Mastery/06-AWS/Labs/EC2-Deployment-Lab.md
---

# Lab — Deploying Taskmanager to AWS EC2

<!-- nav -->
↑ [[AWS-MOC|AWS]] · ← [[6.2-AWS-Deployment-EC2-ECS|AWS Deployment — EC2 & ECS]]
<!-- /nav -->

## What Was Built

- IAM user `devops-admin` with AdministratorAccess
- S3 bucket for storage (`devops-taskmanager-956179206096`)
- EC2 key pair (`devops-key`)
- Security group with ports 22 and 5000 open
- EC2 t2.micro Ubuntu 24.04 instance
- Taskmanager app deployed via Docker Compose on EC2
- Accessed app from local VM hitting EC2 public IP

---

## Resources Created

| Resource | ID/Name |
|----------|---------|
| IAM User | devops-admin |
| S3 Bucket | devops-taskmanager-956179206096 |
| Key Pair | devops-key → `~/.ssh/devops-key.pem` |
| Security Group | sg-062d8ee701e6205f0 (devops-sg) |
| EC2 Instance | devops-lab (stored in txt file) |
| Region | ap-south-1 |

---

## Deployment Flow

```
Local VM
  │ git push
  ▼
GitHub (aakash-1004/taskmanager)
  │ git clone
  ▼
EC2 Instance (13.126.234.226)
  │ docker compose up -d
  ▼
Flask Container (:5000) + MongoDB Container
  │
  ▼
Accessible at http://13.126.234.226:5000
```

---

## Key Observations

**EC2 vs VirtualBox:**
- EC2 has real CPU with AVX support → `mongo:6` runs fine
- VirtualBox VMs lack AVX → must use `mongo:4.4`

**Deployment is just git clone + docker compose:**
- No manual Python setup
- No manual MongoDB install
- Entire stack up in under 2 minutes
- This is the power of containerization

**Security Groups are stateful:**
- Inbound rule for port 5000 allows external access
- Without it, curl from local VM would time out

---

## Cost Management

```bash
# Stop instance (pauses, storage still billed ~$0.10/month)
aws ec2 stop-instances --instance-ids <instance-id>

# Terminate instance (permanent, no more charges)
aws ec2 terminate-instances --instance-ids <instance-id>

# Always stop or terminate when not in use
```

**Free tier limits (monthly):**
- EC2: 750 hours t2.micro
- S3: 5GB storage
- Data transfer: 1GB out

---

## Bugs / Gotchas Hit

| Issue | Cause | Fix |
|-------|-------|-----|
| Docker permission denied | Group change not applied | Exit and re-SSH |
| Can't see instance in console | Wrong region | Switch to ap-south-1 |
| VM lost network (DNS failure) | VirtualBox CPU starvation | Hard reboot VM |
| `InvalidClientTokenId` | Wrong credentials | Re-run `aws configure` |

---

## Related notes
- [[6.1-AWS-Core-Services]]
- [[4.7-Docker-Compose]]
- [[Lab-Taskmanager-Dockerization]]

<!-- related -->
---
**Related:** [[6.1-AWS-Core-Services|AWS Core Services]] · [[AWS-04-EC2|EC2 — Elastic Compute Cloud]] · [[Lab-Taskmanager-Dockerization|Lab — Dockerizing the Taskmanager App]] · [[7.2-GitHub-Actions-Pipeline|GitHub Actions CI/CD Pipeline]]
<!-- /related -->
