---
title: 06 — Data Sources
type: concept
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - data-sources
  - aws
  - brownfield
config: tf-data-sources/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[05-Functions-Expressions-and-Loops]]"
next: "[[07-Count-vs-ForEach]]"
origin: Terraform-Mastery/06-Data-Sources.md
---

# 06 — Data Sources

← Back to [[Terraform-Mastery-MOC]]

---

## Concept

A data source **reads existing infrastructure** that Terraform doesn't manage. It creates nothing, destroys nothing, and appears in state only as a cached read.

Two situations where it's essential:
1. **Dynamic values** that shouldn't be hardcoded — the latest AMI, the list of available AZs, the current account ID
2. **Brownfield integration** — your Terraform config needs to attach to a VPC, subnet, or security group that another team or an older stack owns

> **Memory hook:** `resource` = "make this exist". `data` = "tell me about something that already exists". Look, don't touch.

---

## Config built — `tf-data-sources/`

```hcl
# Latest Amazon-owned AMI
data "aws_ami" "name" {
  most_recent = true
  owners      = ["amazon"]
}

# Existing security group, found by tags
data "aws_security_group" "name" {
  tags = {
    name = "MySG"
    ENV  = "PROD"
  }
}

# Existing VPC, found by tags
data "aws_vpc" "name" {
  tags = {
    ENV  = "PROD"
    Name = "my-vpc"
  }
}

# All available AZs in the current region
data "aws_availability_zones" "names" {
  state = "available"
}

# Subnet inside that VPC, found by filter + tag
data "aws_subnet" "name" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.name.id]     # data source referencing another data source
  }
  tags = {
    Name = "private-subnet"
  }
}

# Who am I / where am I — no arguments needed
data "aws_caller_identity" "name" {}
data "aws_region" "name" {}

# Consume them
resource "aws_instance" "myserver" {
  ami             = "ami-01b1be742d950fb7f"
  instance_type   = "t3.nano"
  subnet_id       = data.aws_subnet.name.id
  security_groups = [data.aws_security_group.name.id]

  tags = { Name = "SampleServer" }
}

output "aws_ami"        { value = data.aws_ami.name.id }
output "aws_zones"      { value = data.aws_availability_zones.names }
output "caller_info"    { value = data.aws_caller_identity.name }
output "region_name"    { value = data.aws_region.name }
output "security_group" { value = data.aws_security_group.name.id }
output "vpc_id"         { value = data.aws_vpc.name.id }
```

**Reference prefix:** `data.<TYPE>.<NAME>.<ATTR>` — the leading `data.` is what distinguishes it from a resource reference.

---

## The AMI lookup — do it properly

The course version is loose:
```hcl
data "aws_ami" "name" {
  most_recent = true
  owners      = ["amazon"]
}
```
This returns *whatever* the most recent Amazon-owned AMI happens to be — could be anything, including an AMI for the wrong architecture or OS. **Never use this shape in real work.** Always filter:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux.id   # no hardcoded, region-locked AMI ID
}
```

Ubuntu equivalent (note the Canonical owner ID — worth memorising, `099720109477`):
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

**The tradeoff to mention in an interview:** `most_recent = true` means the AMI ID can change between plans, which triggers instance replacement when AWS publishes a new image. In production you either pin a specific AMI, or use a Packer-built golden AMI referenced by a stable tag, or accept the churn deliberately because instances are immutable and behind an ASG. Knowing this tradeoff is the difference between "I saw it in a tutorial" and "I've run it".

---

## Useful data sources beyond the course

| Data source | Returns | Common use |
|---|---|---|
| `aws_caller_identity` | account_id, arn, user_id | Building ARNs dynamically without hardcoding the account number |
| `aws_region` | current region name | Region-aware naming and ARN construction |
| `aws_availability_zones` | list of AZ names | Spreading subnets across AZs without hardcoding |
| `aws_ami` | AMI ID | Avoid region-locked hardcoded AMIs |
| `aws_vpc` / `aws_subnet` / `aws_subnets` | existing network IDs | Brownfield: deploy into a VPC someone else owns |
| `aws_iam_policy_document` | JSON policy | Cleaner than `jsonencode` for complex policies |
| `aws_secretsmanager_secret_version` | secret value | Pull secrets at plan time instead of storing them in tfvars |
| `terraform_remote_state` | another state file's outputs | Cross-stack wiring — network stack → app stack |

**`aws_caller_identity` in practice:**
```hcl
policy_arn = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:policy/MyPolicy"
```
Same config now works in any account.

**`terraform_remote_state`** deserves special mention — it's how large orgs split state:
```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tfstate"
    key    = "network/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_id
}
```
The network team owns one state file; the app team reads its outputs without being able to modify it. → [[08-State-Management-and-Remote-Backend]]

---

## Data source vs import — the distinction they test

| | `data` source | `terraform import` |
|---|---|---|
| Terraform manages it? | **No** — read-only | **Yes** — takes ownership |
| Appears in state? | Cached read only | Full resource entry |
| `destroy` affects it? | No | **Yes, it gets destroyed** |
| Use when | Another team/stack owns it | You're adopting existing infra into your config |

> **One-liner:** data source = borrow. Import = adopt.

This comes up constantly in brownfield-migration questions → [[09-State-Manipulation-and-Import]]

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| `aws_ami` with filters | Standard — nobody hardcodes AMI IDs in a config that deploys to more than one region |
| `aws_vpc` / `aws_subnet` by tag | The normal situation: a networking team owns the VPC, application teams deploy into it |
| `aws_availability_zones` | Multi-AZ subnet creation without hardcoding `["ap-south-1a", "ap-south-1b"]` |
| `aws_caller_identity` | Account-agnostic ARN construction; also used to assert you're applying to the right account |
| `terraform_remote_state` | Cross-stack dependencies in a split-state architecture |

**Connection to my own work:** the EKS/IRSA setup in `gitops-go-app` needs the OIDC provider URL and account ID to build the trust policy. `aws_caller_identity` plus the EKS cluster data source is exactly how you avoid hardcoding those.

---

## Interview Q&A

**Q: What's a data source and how is it different from a resource?**
> A resource is something Terraform creates and manages — it's in state, and destroy will delete it. A data source is a read-only lookup of something that already exists; Terraform queries it, caches the result, and uses it, but never modifies or deletes it. The two main uses are avoiding hardcoded values, like looking up the latest AMI instead of pasting an ID that's only valid in one region, and integrating with infrastructure another team owns — my application config reads the VPC and subnet IDs by tag rather than trying to manage them.

**Q: How do you avoid hardcoding AMI IDs?**
> A `data "aws_ami"` block with `most_recent = true`, an owner filter, and a name filter matching the image family — for example `al2023-ami-*-x86_64` for Amazon Linux 2023. That makes the config portable across regions, since AMI IDs are region-scoped. The tradeoff is that `most_recent` means the ID can change between plans and trigger instance replacement, so for production I'd either pin a specific AMI, or use a Packer-built golden image referenced by a stable tag, and only accept the churn where instances are genuinely immutable and sit behind an autoscaling group.

**Q: Your application stack needs a subnet ID from a VPC another team manages. How do you wire that up?**
> Two options depending on whether that team uses Terraform. If they do and their state is accessible, I use a `terraform_remote_state` data source to read their published outputs — that's the cleanest because the contract is explicit and versioned. If they don't, or I only have read access to AWS, I use an `aws_subnet` or `aws_subnets` data source filtering by tag. Either way it's a data source, not an import, because I don't want ownership — if I imported it, my destroy would delete their VPC.

**Q: Do data sources cost you anything at plan time?**
> They're read API calls, so each one is a round trip to AWS on every plan and refresh. That's usually negligible, but in a large config with dozens of data sources it's a noticeable chunk of plan time, and it means plan requires read permissions on everything you look up. It's also a failure mode worth knowing: if a data source's lookup returns zero results or more than one where exactly one is expected, the plan fails outright rather than degrading gracefully.

---

## Gotchas & flags

- A data source that matches **zero** resources fails the plan. One that matches **multiple** where a singular type is used (`aws_subnet` vs `aws_subnets`) also fails. Filter precisely.
- `most_recent = true` without name filters is effectively a random AMI. The course config does this — don't copy it.
- Data sources are re-read on **every** plan and refresh. They're not cached across runs.
- Data sources can reference other data sources (`aws_subnet` filtering on `data.aws_vpc.name.id`) — Terraform orders them in the dependency graph automatically.
- `data.aws_caller_identity` and `data.aws_region` take no arguments — empty braces are correct, not a mistake.

---

Next: [[07-Count-vs-ForEach]]
