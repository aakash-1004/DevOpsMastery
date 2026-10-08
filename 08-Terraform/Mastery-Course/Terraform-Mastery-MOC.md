---
title: Terraform Mastery — Map of Content
aliases:
  - 00-Terraform-MOC
type: moc
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - iac
  - aws
  - devops
  - moc
up: "[[Terraform-MOC]]"
related:
  - "[[8.1-Terraform-Core-Concepts]]"
  - "[[8.2-Terraform-AWS-Infrastructure-Project]]"
origin: Terraform-Mastery/00-Terraform-MOC.md
course: Prashant Kumar Paradkar — Terraform + AWS (9h) + hands-on configs
---

# Terraform Mastery — Map of Content

<!-- nav -->
↑ [[Terraform-MOC|Terraform]]
<!-- /nav -->

> **One-glance revision hub.** Every concept from the 9h course, mapped to the config that proves it, plus the interview question it answers.

---

## Revision path (read in this order)

| # | Note | Covers | Interview weight |
|---|---|---|---|
| 01 | [[01-Terraform-Fundamentals-and-Workflow]] | IaC, HCL, init/validate/plan/apply/destroy | ★★★★★ |
| 02 | [[02-HCL-Block-Types]] | All 8 block types, when each is used | ★★★★☆ |
| 03 | [[03-Providers-and-First-Resource-EC2]] | provider config, versioning, first resource | ★★★★☆ |
| 04 | [[04-Variables-Deep-Dive]] | types, precedence, tfvars, validation, locals | ★★★★★ |
| 05 | [[05-Functions-Expressions-and-Loops]] | built-in functions, for expressions, operators | ★★★☆☆ |
| 06 | [[06-Data-Sources]] | reading existing infra, killing hardcoded IDs | ★★★★☆ |
| 07 | [[07-Count-vs-ForEach]] | multi-resource creation, the classic trap | ★★★★★ |
| 08 | [[08-State-Management-and-Remote-Backend]] | tfstate, S3 backend, locking, drift | ★★★★★ |
| 09 | [[09-State-Manipulation-and-Import]] | state mv/rm/list, brownfield import | ★★★★★ |
| 10 | [[10-Modules]] | registry modules, building + publishing own | ★★★★★ |
| 11 | [[11-Dependencies-Lifecycle-Validations]] | depends_on, lifecycle, pre/postconditions | ★★★★☆ |
| 12 | [[12-Workspaces]] | multi-env in one config dir, and its limits | ★★★☆☆ |
| 13 | [[13-AWS-Networking-VPC-Concepts]] | region/AZ/subnet/CIDR/IGW/NAT/SG/NACL | ★★★★★ |
| 14 | [[14-Project-S3-Static-Website]] | full S3 static hosting build | ★★★☆☆ |
| 15 | [[15-Project-VPC-EC2-Nginx]] | network + compute + user_data end to end | ★★★★☆ |
| 16 | [[16-Project-IAM-YAML-Automation]] | yamldecode + flatten + for_each at scale | ★★★★☆ |
| 17 | [[17-Terraform-Cloud-and-Best-Practices]] | TFC, CI/CD, security, repo hygiene | ★★★★☆ |
| 18 | [[18-Interview-Question-Bank]] | consolidated rapid-fire Q&A | ★★★★★ |

---

## The 6 concepts that get asked in every single interview

Everything else is secondary. If time is short, revise only these:

1. **`plan` vs `apply` and why state exists** → [[01-Terraform-Fundamentals-and-Workflow]]
2. **State drift — what it is, how you detect and fix it** → [[08-State-Management-and-Remote-Backend]]
3. **Remote state + locking (S3 + DynamoDB)** → [[08-State-Management-and-Remote-Backend]]
4. **`count` vs `for_each` and why `for_each` is safer** → [[07-Count-vs-ForEach]]
5. **Module design — inputs, outputs, reuse** → [[10-Modules]]
6. **`terraform import` for brownfield infra** → [[09-State-Manipulation-and-Import]]

---

## Config inventory — what was built

| Directory | Concept proved |
|---|---|
| `aws-ec2/` | first resource, variables, outputs split |
| `aws-s3/` | multi-provider (aws + random), resource chaining |
| `tf-backend/` | S3 remote backend |
| `proj-static-website/` | S3 static hosting, bucket policy, jsonencode |
| `aws-vpc/` | VPC, subnets, IGW, route tables from scratch |
| `aws-vpc-ec2-nginx/` | full stack + user_data bootstrap + SG |
| `tf-data-sources/` | aws_ami, aws_vpc, aws_subnet, caller_identity lookups |
| `tf-variables/` | object types, validation, tfvars precedence, merge() |
| `tf-functions/` | built-in function catalogue |
| `tf-operators-exps/` | for expressions over list/map/object |
| `tf-multi-resources/` | count + for_each side by side |
| `aws-IAM-management/` | yamldecode, flatten, dynamic IAM at scale |
| `tf-module-vpc/` | consuming public registry modules |
| `tf-own-module-VPC/` | authoring a module with validation + optional() |
| `testing-local-module/` | consuming a published registry module |
| `tf-test/` | depends_on, lifecycle, precondition, postcondition |
| `tf-cli-workspace/` | workspace-scoped resource naming |

---

## How this connects to my existing work

- **`terraform-fullstack`** (my repo) — already does VPC/ECS/Fargate/ALB via reusable modules with S3+DynamoDB remote state. This course formalises *why* those choices are correct → [[08-State-Management-and-Remote-Backend]], [[10-Modules]]
- **`gitops-go-app`** — ArgoCD/Helm/EKS. Terraform provisions the cluster layer beneath it; Helm/ArgoCD handle the workload layer. Know the boundary → [[17-Terraform-Cloud-and-Best-Practices]]
- **JS7/Keycloak production support** — the debugging instinct transfers directly. Drift in Terraform is the same failure class as "config file says X, running process says Y" → [[08-State-Management-and-Remote-Backend]]

---

## Gaps to close next

- [ ] `terraform import` run hands-on against a manually-created EC2 (only read about it, not executed)
- [ ] DynamoDB state locking actually demonstrated (course backend config omits it — this is a real gap in the course)
- [ ] Terraform Cloud workspace connected to a GitHub repo
- [ ] `terraform plan -out=tfplan` + `terraform apply tfplan` in a CI pipeline
- [ ] `checkov` / `tfsec` policy scanning on own configs

<!-- related -->
---
**Related:** [[8.1-Terraform-Core-Concepts|Terraform Core Concepts]] · [[8.2-Terraform-AWS-Infrastructure-Project|Terraform — Infrastructure as Code]]
<!-- /related -->
