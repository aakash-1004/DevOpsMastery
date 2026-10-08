---
title: Lab — Terraform AWS Infrastructure
aliases:
  - Terraform-AWS-Lab
type: lab
module: terraform
status: ✅ Completed
tags:
  - terraform
  - aws
  - lab
  - day5
  - golden-thread
repo: "https://github.com/aakash-1004/terraform-aws (private)"
up: "[[Terraform-MOC]]"
prev: "[[8.2-Terraform-AWS-Infrastructure-Project]]"
related:
  - "[[8.1-Terraform-Core-Concepts]]"
origin: DevOps-Mastery/08-Terraform/Labs/Terraform-AWS-Lab.md
---

# Lab — Terraform AWS Infrastructure

<!-- nav -->
↑ [[Terraform-MOC|Terraform]] · ← [[8.2-Terraform-AWS-Infrastructure-Project|Terraform — Infrastructure as Code]]
<!-- /nav -->

## What Was Built

Provisioned complete AWS infrastructure using Terraform:
- S3 bucket (imported existing)
- Security group (ports 22, 5000)
- EC2 t3.micro instance with user_data bootstrap
- App deployed automatically on boot via user_data
- Full destroy in 20 seconds

---

## Files

```
terraform-aws/
├── main.tf
├── variables.tf
├── outputs.tf
├── .terraform.lock.hcl
└── .gitignore
```

---

## main.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_s3_bucket" "taskmanager_bucket" {
  bucket = "taskmanager-terraform-${var.account_id}"
  tags = {
    Name        = "taskmanager-bucket"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_security_group" "taskmanager_sg" {
  name        = "taskmanager-sg-tf"
  description = "Taskmanager security group"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 5000
    to_port     = 5000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "taskmanager-sg"
    ManagedBy = "terraform"
  }
}

resource "aws_instance" "taskmanager_server" {
  ami                    = "ami-0f11fb0f6d8b520d4"
  instance_type          = "t3.micro"
  key_name               = "devops-key"
  vpc_security_group_ids = [aws_security_group.taskmanager_sg.id]

  user_data = <<-EOF
    #!/bin/bash
    apt update -y
    apt install -y docker.io docker-compose-v2
    usermod -aG docker ubuntu
    systemctl enable docker
    systemctl start docker
    cd /home/ubuntu
    git clone https://github.com/aakash-1004/taskmanager.git
    cd taskmanager
    docker compose up -d
  EOF

  tags = {
    Name      = "taskmanager-server"
    ManagedBy = "terraform"
  }
}
```

---

## variables.tf

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "account_id" {
  description = "AWS account ID"
  type        = string
  default     = "956179206096"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

---

## outputs.tf

```hcl
output "bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.taskmanager_bucket.bucket
}

output "bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.taskmanager_bucket.arn
}

output "ec2_public_ip" {
  description = "EC2 instance public IP"
  value       = aws_instance.taskmanager_server.public_ip
}
```

---

## Commands Used

```bash
terraform init
terraform plan
terraform apply
terraform import aws_s3_bucket.taskmanager_bucket taskmanager-terraform-956179206096
terraform destroy
```

---

## Key Lessons

**Destroy order is automatic:**
- Terraform destroys in reverse dependency order
- EC2 and S3 destroyed in parallel first
- Security group destroyed last (EC2 depended on it)

**user_data = zero-touch deployment:**
- App was running by the time `terraform apply` finished
- No SSH required
- This is how production deployments work at scale

**import existing resources:**
- Bucket existed from manual creation
- `terraform import` brought it under Terraform management
- `terraform plan` showed `No changes` after import

**State is critical:**
- After destroy, `terraform.tfstate` shows empty resources
- Re-running `terraform apply` recreates everything from scratch

---

## Bugs / Gotchas Hit

| Issue | Cause | Fix |
|-------|-------|-----|
| `t2.micro` not eligible | Account free tier type | Use `t3.micro` |
| Bucket already exists | Created on devops-labs earlier | `terraform import` |
| 674MB file in Git | `.terraform/` committed | `git filter-branch` to rewrite history |
| `terraform.tfstate` committed | Added with `git add .` | `git rm --cached`, add to `.gitignore` |

---

## Related notes
- [[8.1-Terraform-Core-Concepts]]
- [[6.1-AWS-Core-Services]]
- [[Lab-EC2-Deployment]]

<!-- related -->
---
**Related:** [[8.1-Terraform-Core-Concepts|Terraform Core Concepts]]
<!-- /related -->
