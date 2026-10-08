---
title: Revision — Terraform (IaC)
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
order: 7
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-06-AWS-Core-Services]]"
next: "[[Revision-08-CICD-GitHub-Actions-and-Jenkins]]"
related:
  - "[[Terraform-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Terraform (IaC)

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-06-AWS-Core-Services|Revision — AWS Core Services]] · → [[Revision-08-CICD-GitHub-Actions-and-Jenkins|Revision — CI/CD: GitHub Actions & Jenkins]]
<!-- /nav -->

## Concept

Terraform is Infrastructure as Code — you define your infrastructure in HCL (HashiCorp Configuration Language), and Terraform creates/updates/destroys it to match. The critical concept is **state**: Terraform tracks what it has created in a state file (`terraform.tfstate`). This is how it knows what to update vs. create vs. destroy.

**Workflow:**

```
Write HCL → terraform init → terraform plan → terraform apply → terraform destroy
```

---

## Core Syntax & Commands

```bash
terraform init           # Initialize: download providers, set up backend
terraform plan           # Preview changes (what will be created/updated/destroyed)
terraform apply          # Apply changes (prompts for confirmation)
terraform apply -auto-approve  # Skip confirmation (use in CI/CD)
terraform destroy        # Destroy all managed infrastructure
terraform fmt            # Format HCL files to canonical style
terraform validate       # Validate HCL syntax
terraform output         # Show output values
terraform state list     # List all resources in state
terraform state show aws_instance.web  # Show state for specific resource
terraform import aws_s3_bucket.my_bucket mybucket-name  # Import existing resource
```

**Provider:**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # Allow 5.x.x but not 6.x.x
    }
  }
  backend "s3" {
    bucket  = "tf-state-aakash"
    key     = "prod/terraform.tfstate"
    region  = "ap-south-1"
    encrypt = true
  }
}

provider "aws" {
  region = var.aws_region
}
```

**Resource, Variable, Output:**

```hcl
# variables.tf
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

# main.tf
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id

  tags = {
    Name        = "web-server"
    Environment = var.env
  }

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y docker.io
    systemctl start docker
    systemctl enable docker
  EOF
}

# outputs.tf
output "web_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
}
```

**Remote State (S3 Backend) — Critical for team use:**

```hcl
backend "s3" {
  bucket         = "my-terraform-state"
  key            = "project/env/terraform.tfstate"
  region         = "ap-south-1"
  encrypt        = true
  dynamodb_table = "terraform-state-lock"  # Prevents concurrent applies
}
```

---

## Senior Interview Q&A

**Q: What is Terraform state and why is it important?**

> "Terraform state is the source of truth for what Terraform has deployed. It maps your HCL resource definitions to real infrastructure IDs — 'this `aws_instance.web` is EC2 instance `i-0abc12345`. '
>
> Without state, Terraform would have to query every AWS resource on every plan to figure out what exists. That would be slow and AWS API rate-limited. State makes plans fast.
>
> State also tracks dependencies — so if you delete a VPC resource, Terraform knows to delete its subnets first.
>
> The critical concern: **state contains secrets in plaintext**. Database passwords, private keys — they're all in the state file. This is why you never commit state to Git. Store it in S3 with encryption enabled, and use DynamoDB for state locking (prevents two people running `terraform apply` simultaneously, which would corrupt state).
>
> In practice at teams: S3 backend + DynamoDB lock is the minimum setup. Some teams use Terraform Cloud for state management with built-in locking and audit trails."

---

**Q: What's `terraform plan` doing under the hood?**

> "Plan compares three things: your HCL code (desired state), the Terraform state file (last known real state), and the actual infrastructure (refreshed from the provider API).
>
> It runs a refresh first — calls AWS APIs to check if state-tracked resources still exist and match. If you manually deleted something in the console, plan detects the drift.
>
> Then it diffs desired state vs. current state and produces a set of actions: create (resource in code, not in state), update (resource exists but attributes differ), destroy (resource in state, not in code), replace (forced replacement — like changing an AMI which requires EC2 to be destroyed and recreated — shown as `-/+`).
>
> The `~` symbol means in-place update. The `-/+` means destroy-then-create. Always review `-/+` carefully — that means downtime or data loss risk.
>
> I always run plan in CI before apply, and the plan output is attached to the PR for review. Nobody applies without a reviewed plan."

---

**Q: How do you handle sensitive values in Terraform?**

> "Three approaches, in order of preference:
>
> 1. **Don't put secrets in Terraform at all.** For RDS passwords, use AWS Secrets Manager and pass the ARN to the resource. `password = data.aws_secretsmanager_secret_version.db.secret_string`. The secret never touches Terraform state.
>     
> 2. **Use `sensitive = true` on variables.** This prevents the value from appearing in `terraform plan` output. But note: it's still in the state file — just not shown in logs.
>     
> 3. **Pass secrets at runtime via environment variables.** `export TF_VAR_db_password=$SECRET`. The variable is in Terraform but never committed to source control.
>     
>
> What I never do: put secrets in `terraform.tfvars` and commit that file to Git. I've seen teams do this and it's a major security incident waiting to happen. `.gitignore` must include `*.tfvars` files containing secrets."

<!-- related -->
---
**Related:** [[Terraform-MOC|Terraform]]
<!-- /related -->
