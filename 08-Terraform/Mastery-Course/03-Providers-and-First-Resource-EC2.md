---
title: "03 — Providers & First Resource (EC2)"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - provider
  - ec2
  - aws
  - project-structure
config: aws-ec2/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[02-HCL-Block-Types]]"
next: "[[04-Variables-Deep-Dive]]"
origin: Terraform-Mastery/03-Providers-and-First-Resource-EC2.md
---

# 03 — Providers & First Resource (EC2)

← Back to [[Terraform-Mastery-MOC]]

---

## Config built — `aws-ec2/`

Three-file split. This is the canonical layout and what interviewers expect when they ask "how do you structure a Terraform project".

**`main.tf`**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.54.1"
    }
  }
}

provider "aws" {
  region = var.region
}

resource "aws_instance" "myserver" {
  ami           = "ami-0c0e147c706360bd7"
  instance_type = "t3.nano"

  tags = {
    Name = "SampleServer"
  }
}
```

**`variables.tf`**
```hcl
variable "region" {
  description = "Value of region"
  type        = string
  default     = "eu-north-1"
}
```

**`outputs.tf`**
```hcl
output "aws_instance_public_ip" {
  value = aws_instance.myserver.public_ip
}
```

---

## Standard file layout

| File | Contains |
|---|---|
| `main.tf` | provider config + primary resources |
| `variables.tf` | all `variable` blocks |
| `outputs.tf` | all `output` blocks |
| `terraform.tfvars` | actual values — **often gitignored** |
| `versions.tf` | `terraform {}` block (larger projects split this out of main.tf) |
| `locals.tf` | `locals` blocks, once there are enough to warrant it |

**Important:** Terraform loads **every `.tf` file in the directory** and concatenates them. File names carry zero meaning to Terraform — they're purely for human organisation. `aws-vpc-ec2-nginx/` proves this: it splits into `vpc.tf`, `ec2.tf`, `security_groups.tf`, `providers.tf`, `outputs.tf`, and resources in `ec2.tf` reference resources in `vpc.tf` with no imports needed.

> **Memory hook:** Terraform sees one big file. You see many. Split by *domain* (network, compute, security), not by block type, once a project gets real.

---

## AWS CLI profile setup (prerequisite)

```bash
aws configure --profile devops-admin     # writes creds + region + output format
aws sts get-caller-identity --profile devops-admin   # verify WHO you are — always do this first
aws configure list                       # shows values AND where each is sourced from
```

Files written (Windows: `C:\Users\<you>\.aws\`):

```ini
# credentials
[devops-admin]
aws_access_key_id     = AKIA...
aws_secret_access_key = ...

# config
[profile devops-admin]     # NOTE: "profile " prefix required here, NOT in credentials
region = ap-south-1
output = json
```

Wire it into Terraform:
```hcl
provider "aws" {
  region  = "ap-south-1"
  profile = "devops-admin"
}
```

Or set it per shell session:
```powershell
$env:AWS_PROFILE="devops-admin"   # PowerShell
export AWS_PROFILE=devops-admin   # bash
```

**Why named profiles matter for interviews:** every company with more than one AWS account (dev/staging/prod, or separate accounts per team) runs multi-profile. "How do you manage multiple AWS accounts locally?" → named profiles in `~/.aws/config`, plus `assume_role` blocks in the provider for cross-account access.

---

## Cross-account access via assume_role

Worth knowing even though the course didn't cover it — it's the next question after "named profiles":

```hcl
provider "aws" {
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformExecutionRole"
    session_name = "terraform"
  }
}
```

This is how the real pattern works: one identity account holds users, workload accounts hold roles, Terraform assumes into the target account.

---

## Key EC2 resource arguments

| Argument | Notes |
|---|---|
| `ami` | Region-specific. An AMI ID valid in `eu-north-1` is invalid in `ap-south-1`. **Hardcoding this is the #1 beginner smell** → fix with `data "aws_ami"` in [[06-Data-Sources]] |
| `instance_type` | `t3.micro` is the current free-tier default for new accounts; `t2.micro` for older ones. Verify per account. |
| `subnet_id` | If omitted, lands in the default VPC's default subnet |
| `vpc_security_group_ids` | List of SG IDs. If omitted, uses the VPC default SG — **which typically has no inbound rules, making the instance unreachable** |
| `key_name` | SSH key pair name. If omitted, **no SSH access at all, ever** — you can't add it after launch |
| `associate_public_ip_address` | Explicitly control public IP assignment |
| `user_data` | Bootstrap script run at first boot → [[15-Project-VPC-EC2-Nginx]] |
| `root_block_device` | Nested block for volume size/type/encryption |
| `tags` | Map. `Name` is the one that renders in the console. |

---

## Reading the plan output

From an actual apply during this course:

```
+ resource "aws_instance" "myEc2" {
    + ami           = "ami-01a00762f46d584a1"
    + instance_type = "t3.micro"
    + id            = (known after apply)
    + public_ip     = (known after apply)
    + key_name      = (known after apply)      ← no key pair set = no SSH, ever
    + vpc_security_group_ids = (known after apply)  ← falls back to default SG
  }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**What this plan is telling you that's easy to miss:** `key_name` and `vpc_security_group_ids` unset means the instance boots, gets a public IP, outputs cleanly — and is completely unreachable. Provisioning success ≠ usable infrastructure.

Plan symbols:
- `+` create
- `-` destroy
- `~` update in place (no downtime)
- `-/+` destroy then create (**replacement — watch for this, it means downtime**)
- `<=` read (data source refresh)

> **The symbol to fear is `-/+`.** Always check *why* something is being replaced. Terraform prints `# forces replacement` next to the offending attribute.

---

## Real-world mapping

| Course concept | Job reality |
|---|---|
| Hardcoded AMI | Replaced by `data "aws_ami"` filter, or a Packer-built golden AMI referenced by tag |
| `region` as a variable | Region comes from the environment's tfvars; same config deploys to ap-south-1 and us-east-1 |
| Profile in provider block | In CI it's an IAM role via OIDC, not a profile — you never ship a long-lived key into a pipeline |
| Raw `aws_instance` | Production rarely creates bare instances — it's ASGs + launch templates, or ECS/EKS. Bare EC2 is for bastion hosts and one-offs. |

---

## Interview Q&A

**Q: How do you structure a Terraform project?**
> At minimum, main.tf for providers and resources, variables.tf for inputs, outputs.tf for outputs, and a tfvars file per environment. Once the project grows I split main.tf by domain rather than by block type — so vpc.tf, ec2.tf, security_groups.tf — because Terraform loads every .tf file in the directory anyway and concatenates them. Beyond that, I move reusable pieces into modules and keep the root module thin, just wiring modules together with environment-specific inputs.

**Q: How do you handle credentials for the AWS provider?**
> Never in the provider block — hardcoded keys end up in Git. Locally I use a named profile in the shared credentials file and reference it with the `profile` argument or the AWS_PROFILE environment variable. In CI I use short-lived credentials from an IAM role, usually via OIDC federation from GitHub Actions, so there's no long-lived secret stored anywhere. For cross-account work I add an `assume_role` block to the provider pointing at a role in the target account.

**Q: You ran apply, it succeeded, the instance is running, but you can't SSH in. Walk me through it.**
> First thing I'd check is whether a key pair was ever attached — if `key_name` wasn't set at launch, there's no way in, and you can't add one retroactively; the instance has to be replaced. Assuming the key is there, I'd check the security group for an inbound rule on port 22 from my IP, then confirm the instance is in a public subnet with a route to an internet gateway and has a public IP assigned. The distinction that narrows it fastest is timeout versus connection refused — a timeout means the packet is being silently dropped, so it's a security group, NACL, or routing problem. Connection refused means it reached the host and nothing is listening, so it's sshd.

**Q: What does `-/+` mean in a plan and why does it matter?**
> It means destroy and recreate rather than update in place. It matters because it's downtime and, for stateful resources, data loss. Terraform annotates the attribute causing it with "forces replacement" — usually something immutable like an AMI change, an availability zone change, or a subnet change on an instance. If I see it on something I didn't expect, I stop and investigate before applying, and if the resource genuinely must be replaced but can't tolerate a gap, I add `create_before_destroy` to its lifecycle block.

---

## Gotchas & flags

- AMI IDs are **region-scoped**. Copying a config between regions without changing the AMI fails at apply, not at validate.
- Changing `ami` on an existing instance forces replacement. Same for `subnet_id` and `availability_zone`.
- `security_groups` (names) vs `vpc_security_group_ids` (IDs) — in a non-default VPC, always use `vpc_security_group_ids`. The `security_groups` argument is a legacy EC2-Classic holdover and causes spurious diffs.
- Free tier: verify `t3.micro` vs `t2.micro` per account — it's not universal.

---

Next: [[04-Variables-Deep-Dive]]
