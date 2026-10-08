---
title: 02 — HCL Block Types
type: concept
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - hcl
  - syntax
  - blocks
up: "[[Terraform-Mastery-MOC]]"
prev: "[[01-Terraform-Fundamentals-and-Workflow]]"
next: "[[03-Providers-and-First-Resource-EC2]]"
origin: Terraform-Mastery/02-HCL-Block-Types.md
---

# 02 — HCL Block Types

← Back to [[Terraform-Mastery-MOC]]

---

## The 8 blocks that matter

| Block | Purpose | Creates infra? |
|---|---|---|
| `terraform {}` | Configures Terraform itself — provider versions, backend, CLI version | No |
| `provider {}` | Which cloud, which region, which credentials | No |
| `resource {}` | The thing being created | **Yes** |
| `variable {}` | Input parameter | No |
| `output {}` | Value returned after apply / exposed to parent module | No |
| `data {}` | Read-only lookup of existing infra | No |
| `locals {}` | Internal computed shortcut | No |
| `module {}` | Reusable bundle of resources | **Yes (indirectly)** |

> **Memory hook:** only `resource` and `module` actually build things. Everything else is plumbing, inputs, or lookups.

---

## 1. `terraform {}` — Terraform's settings drawer

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.54.1"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.6.2"
    }
  }

  backend "s3" {
    bucket = "my-tfstate-bucket"
    key    = "prod/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

- `source = "hashicorp/aws"` → registry address, format `NAMESPACE/NAME`
- `version = "5.54.1"` → exact pin
- `required_version` → pins the Terraform CLI itself
- `backend` → where state lives → [[08-State-Management-and-Remote-Backend]]

**Version constraint syntax — know all of these:**

| Constraint | Meaning |
|---|---|
| `= 5.54.1` or `5.54.1` | exactly this version |
| `>= 5.30` | this or newer — risky, allows major bumps |
| `~> 5.54.1` | allows 5.54.2, blocks 5.55.0 — **patch only** |
| `~> 5.54` | allows 5.55, 5.99, blocks 6.0 — **minor + patch** |
| `>= 5.0, < 6.0` | explicit range |

**Which to use:** `~>` in team repos. It picks up security patches without letting a breaking major version in. Exact pins are for when you need absolute determinism (rare, since `.terraform.lock.hcl` already gives you that).

---

## 2. `provider {}` — which cloud, which account

```hcl
provider "aws" {
  region  = "ap-south-1"
  profile = "devops-admin"
}
```

**Multi-region / multi-account via `alias`:**

```hcl
provider "aws" {
  region = "ap-south-1"          # default provider
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

resource "aws_s3_bucket" "logs" {
  provider = aws.us_east         # explicitly use the aliased one
  bucket   = "my-us-logs"
}
```

**Interview relevance:** "How do you deploy the same infrastructure to two regions?" → provider aliasing, not duplicated configs. Also comes up for ACM certificates, which for CloudFront **must** be created in `us-east-1` regardless of where the rest of the stack lives.

**Credential resolution order** (Terraform uses the AWS SDK chain):
1. Static creds in the provider block — **never do this**, they end up in Git
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
3. Shared credentials file (`~/.aws/credentials`) + `profile` argument
4. EC2 instance profile / ECS task role / EKS IRSA

In CI, options 2 or 4. On a laptop, option 3.

---

## 3. `resource {}` — the actual infrastructure

```hcl
resource "aws_instance" "myserver" {
  ami           = "ami-0c0e147c706360bd7"
  instance_type = "t3.nano"

  tags = {
    Name = "SampleServer"
  }
}
```

Syntax: `resource "<TYPE>" "<LOCAL_NAME>" { ... }`

- **TYPE** (`aws_instance`) — fixed by the provider, you don't invent it. The prefix before the first underscore tells Terraform which provider handles it.
- **LOCAL_NAME** (`myserver`) — arbitrary, chosen by you. It's the handle you use in references: `aws_instance.myserver.public_ip`. **It is not the name in the AWS console** — that comes from the `Name` tag.
- **Resource address** = `aws_instance.myserver`. This is what `terraform state` commands take.

> **Common confusion, worth nailing:** local name ≠ AWS name. Two different namespaces. The local name lives only inside Terraform.

---

## 4. `variable {}` — input

```hcl
variable "region" {
  description = "Value of region"
  type        = string
  default     = "eu-north-1"
}
```

Full coverage in [[04-Variables-Deep-Dive]].

---

## 5. `output {}` — return value

```hcl
output "aws_instance_public_ip" {
  description = "The public IP address of the EC2 instance"
  value       = aws_instance.myserver.public_ip
}
```

Two jobs:
1. Print a value in the terminal after apply
2. **Expose a child module's value to its parent** — `module.vpc.vpc_id`. This is the more important one.

Options: `sensitive = true` (masks it in CLI output — but it's still plaintext in state), `depends_on` (rare, forces ordering).

---

## 6. `data {}` — read, don't touch

```hcl
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
}

# referenced as: data.aws_ami.latest.id
```

Note the reference prefix: `data.` — resources are `aws_instance.x`, data sources are `data.aws_ami.x`. Full coverage in [[06-Data-Sources]].

---

## 7. `locals {}` — internal shortcut

```hcl
locals {
  project = "project-01"
  owner   = "ABC"

  common_tags = {
    Project   = local.project
    ManagedBy = "terraform"
  }
}

# referenced as: local.project   (singular "local", not "locals")
```

**`variable` vs `local` — the distinction they test:**

| | `variable` | `local` |
|---|---|---|
| Set from outside? | Yes (tfvars, CLI, env) | No |
| Can reference other values? | No (defaults must be literal) | Yes — can compute from vars, resources, functions |
| Purpose | Parameterise the config | Avoid repeating an expression |

> **Memory hook:** variables are the *knobs on the outside of the box*; locals are the *wiring inside*.

---

## 8. `module {}` — reusable bundle

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# outputs referenced as: module.vpc.vpc_id
```

Full coverage in [[10-Modules]].

---

## Comments

```hcl
# Preferred style — this is what terraform fmt expects
// Also valid, but out of place in HCL
/*
  Multi-line. Genuinely useful for temporarily
  disabling a whole block during debugging.
*/
```

**Rule: comment the *why*, not the *what*.**

```hcl
instance_type = "t3.micro"   # BAD: "sets instance type"
instance_type = "t3.micro"   # GOOD: free-tier eligible, lab workload only
```

Interviewers who look at your GitHub read comments as evidence you understood the decision rather than copied a tutorial.

---

## Reference syntax cheat sheet

| Referencing | Syntax | Example |
|---|---|---|
| Resource attribute | `<type>.<name>.<attr>` | `aws_vpc.my_vpc.id` |
| Data source | `data.<type>.<name>.<attr>` | `data.aws_ami.latest.id` |
| Variable | `var.<name>` | `var.region` |
| Local | `local.<name>` | `local.project` |
| Module output | `module.<name>.<output>` | `module.vpc.vpc_id` |
| Current workspace | `terraform.workspace` | — |
| Self (in lifecycle) | `self.<attr>` | `self.public_ip` |
| count index | `count.index` | — |
| for_each key/value | `each.key` / `each.value` | — |
| String interpolation | `"${...}"` | `"demo-${random_id.r.hex}"` |

---

## Interview Q&A

**Q: Walk me through the block types in a Terraform configuration.**
> `terraform` configures Terraform itself — provider versions, required CLI version, and the backend. `provider` configures the connection to a specific cloud: region, credentials, and aliases if I need multiple regions. `resource` is the only block that actually creates infrastructure. `variable` and `output` are the inputs and outputs — outputs are especially important because that's how a child module hands values back to its parent. `data` reads existing infrastructure without managing it. `locals` are computed values used internally to avoid repetition. And `module` bundles a group of resources for reuse.

**Q: Difference between a variable and a local?**
> A variable is an input that can be set from outside the configuration — a tfvars file, a CLI flag, or an environment variable — and its default has to be a literal value. A local is computed inside the configuration and can't be overridden externally, but it *can* reference variables, resources, and functions. So I use variables for anything that differs between environments, and locals for derived values like a naming prefix or a common tags map that I want to define once and reuse.

**Q: How do you deploy resources to two different AWS regions in one configuration?**
> Provider aliasing. I declare a second `provider "aws"` block with an `alias` and a different region, then set `provider = aws.us_east` on the resources that need it. The real-world case where this is forced on you is ACM certificates for CloudFront — CloudFront only accepts certificates from us-east-1, so even an entirely ap-south-1 stack needs an aliased us-east-1 provider just for that certificate.

**Q: What's the difference between the resource's local name and its Name tag?**
> The local name is Terraform's internal handle — it's how I reference the resource elsewhere in my code, like `aws_instance.myserver.public_ip`, and it's also the address used by state commands. The Name tag is AWS metadata and is what shows up in the console. They're completely separate namespaces; changing the local name forces a destroy-and-recreate because Terraform sees it as a different resource address, whereas changing the Name tag is just an in-place tag update.

---

Next: [[03-Providers-and-First-Resource-EC2]]
