---
title: 10 — Modules
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - modules
  - registry
  - reusability
config: "tf-module-vpc/, tf-own-module-VPC/, testing-local-module/"
up: "[[Terraform-Mastery-MOC]]"
prev: "[[09-State-Manipulation-and-Import]]"
next: "[[11-Dependencies-Lifecycle-Validations]]"
origin: Terraform-Mastery/10-Modules.md
---

# 10 — Modules

← Back to [[Terraform-Mastery-MOC]]

---

## Concept

A module is **a directory containing `.tf` files**. That's the entire definition. Every Terraform configuration is already a module — the directory you run `apply` in is the **root module**. Anything it calls is a **child module**.

Modules are how you stop copy-pasting resource blocks between environments.

```
root module (where you run terraform apply)
  └── module "vpc"        → child module
  └── module "app"        → child module
        └── module "alb"  → nested child module
```

---

## Consuming a registry module — `tf-module-vpc/`

```hcl
provider "aws" {
  region = "eu-north-1"
}

data "aws_availability_zones" "name" {
  state = "available"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = data.aws_availability_zones.name.names
  public_subnets  = ["10.0.0.0/24"]
  private_subnets = ["10.0.1.0/24"]

  tags = { Name = "test-vpc-module" }
}

module "ec2-instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.6.1"

  name = "single-instance"

  ami                    = "ami-052387465d846f3fc"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_id              = module.vpc.public_subnets[0]

  tags = {
    Name        = "module-project"
    Environment = "dev"
  }
}
```

**Key points:**
- Module outputs are consumed as `module.<name>.<output>` — `module.vpc.public_subnets[0]`
- One module's output feeding another module's input creates an **implicit dependency**; Terraform orders them automatically
- `version` is mandatory in practice for registry modules — without it, a module update can silently change your infrastructure on the next `init -upgrade`
- `terraform-aws-modules/vpc/aws` is the most-used module in the entire ecosystem. Worth knowing by name.

**Source types:**
```hcl
source = "./modules/vpc"                              # local path
source = "terraform-aws-modules/vpc/aws"              # public registry
source = "app.terraform.io/myorg/vpc/aws"             # private registry
source = "git::https://github.com/org/repo.git//modules/vpc?ref=v1.2.0"   # git, pinned by tag
source = "git@github.com:org/repo.git//modules/vpc?ref=v1.2.0"            # git over SSH
```
The `//` in a git source separates repo from subdirectory. `?ref=` pins a tag, branch, or commit — **always pin to a tag**, never track a branch.

---

## Authoring a module — `tf-own-module-VPC/`

The most substantial config in the course. Requirements it implements:
- Accept a CIDR block and create a VPC
- Accept N subnets, each with CIDR + AZ
- A subnet can be marked `public` (default private)
- If any subnet is public → create an IGW and a route table, and associate the public subnets with it

### Structure
```
tf-own-module-VPC/
├── root-main.tf              # the caller
├── root-outputs.tf
├── module/vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
└── examples/complete/        # required for registry publication
    ├── main.tf
    ├── outputs.tf
    └── README.md
```

### `module/vpc/variables.tf` — the interface
```hcl
variable "vpc_config" {
  description = "To get the CIDR and Name of VPC from user"
  type = object({
    cidr_block = string
    name       = string
  })
  validation {
    condition     = can(cidrnetmask(var.vpc_config.cidr_block))
    error_message = "Invalid CIDR Format - ${var.vpc_config.cidr_block}"
  }
}

variable "subnet_config" {
  description = "Get the CIDR and AZ for the subnets"
  type = map(object({
    cidr_block = string
    az         = string
    public     = optional(bool, false)      # optional with default
  }))
  validation {
    condition     = alltrue([for config in var.subnet_config : can(cidrnetmask(config.cidr_block))])
    error_message = "Invalid CIDR Format"
  }
}
```

**This is the part worth studying.** Two object variables rather than eight flat strings, `optional()` so callers only specify what they care about, and validation that fails at plan time with a readable message. That's a well-designed module interface.

### `module/vpc/main.tf` — the logic
```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_config.cidr_block
  tags = { Name = var.vpc_config.name }
}

resource "aws_subnet" "main" {
  vpc_id   = aws_vpc.main.id
  for_each = var.subnet_config

  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az

  tags = { Name = each.key }
}

locals {
  public_subnet = {
    for key, config in var.subnet_config : key => config if config.public
  }
  private_subnet = {
    for key, config in var.subnet_config : key => config if !config.public
  }
}

# IGW only if at least one public subnet exists
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  count  = length(local.public_subnet) > 0 ? 1 : 0
}

resource "aws_route_table" "main" {
  count  = length(local.public_subnet) > 0 ? 1 : 0
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main[0].id
  }
}

resource "aws_route_table_association" "main" {
  for_each = local.public_subnet

  subnet_id      = aws_subnet.main[each.key].id
  route_table_id = aws_route_table.main[0].id
}
```

**Three techniques combined here, all interview-relevant:**
1. `for_each` over a map → subnets addressed by name, not index
2. `for ... if` in locals → split one input map into public/private without duplicating the input
3. `count = condition ? 1 : 0` → conditional resource creation, referenced as `[0]`

### `module/vpc/outputs.tf` — shaping the return value
```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

locals {
  public_subnet_output = {
    for key, config in local.public_subnet : key => {
      subnet_id = aws_subnet.main[key].id
      az        = aws_subnet.main[key].availability_zone
    }
  }
  private_subnet_output = {
    for key, config in local.private_subnet : key => {
      subnet_id = aws_subnet.main[key].id
      az        = aws_subnet.main[key].availability_zone
    }
  }
}

output "public_subnets"  { value = local.public_subnet_output }
output "private_subnets" { value = local.private_subnet_output }
```

Note: it returns a **map keyed by subnet name** with both ID and AZ, rather than a bare list of IDs. The caller can do `module.vpc.public_subnets["public_subnet-1"].subnet_id`. **Designing the output shape is as much of the module's interface as the inputs are** — a flat list forces callers to rely on positional ordering, which is brittle.

### `module/vpc/versions.tf`
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.30"
    }
  }
}
```

**Module version constraints should be *loose* (`>=`), root module constraints *tight* (`~>` or exact).** A module pinning an exact provider version makes it uncombinable with any other module that pins differently. This is a genuinely good design detail to mention in an interview.

### `root-main.tf` — the caller
```hcl
module "vpc" {
  source = "./module/vpc"

  vpc_config = {
    cidr_block = "10.0.0.0/16"
    name       = "my-test-vpc"
  }
  subnet_config = {
    public_subnet-1 = { cidr_block = "10.0.0.0/24", az = "eu-north-1a", public = true }
    public_subnet-2 = { cidr_block = "10.0.2.0/24", az = "eu-north-1a", public = true }
    private_subnet  = { cidr_block = "10.0.1.0/24", az = "eu-north-1b" }   # public defaults to false
  }
}
```

---

## Publishing to the Terraform Registry

Checklist from the course:
- `README.md` with overview, usage example, inputs/outputs
- `LICENSE` file
- `examples/` directory with at least one working example
- Push to GitHub, repo named **`terraform-<PROVIDER>-<NAME>`** (e.g. `terraform-aws-vpc`) — the naming convention is enforced
- Tag releases with **semantic versions** (`v1.0.0`)
- Connect the repo in the Terraform Registry

Consuming the published result (`testing-local-module/`):
```hcl
module "mprashant-test-vpc" {
  source  = "paulgitrepo/mprashant-test-vpc/aws"
  version = "1.0.0"
  ...
}
```

**Real-world note:** most companies use a **private registry** (Terraform Cloud) or plain Git sources with tag pinning, not the public registry. The mechanics are identical.

**Interview value:** having published a module is a genuine differentiator. Very few candidates have. Even the local module with proper README, examples, validation, and outputs is worth walking through when asked "tell me about something you built."

---

## Module design principles

| Principle | Why |
|---|---|
| One module, one logical concern | A "VPC" module makes sense; a "whole company infra" module doesn't |
| Object variables over many flat strings | Related values stay related; interface stays readable |
| `optional()` with sensible defaults | Callers specify only what they care about |
| Validate inputs | Fail at plan with a clear message, not mid-apply with an API error |
| Output shape is interface | Return maps keyed by name, not positional lists |
| Loose provider constraints in modules | Otherwise modules can't be combined |
| No `provider` blocks inside modules | Providers are configured by the root module and inherited. A provider block in a module makes it uncombinable and is deprecated. |
| No hardcoded environment values | No region, no account ID, no environment name baked in |
| README with a working usage example | If it isn't documented, nobody will use it |

> **The provider-block rule is a real interview question.** Modules should *declare* provider requirements in `required_providers` but never *configure* providers. Configuration is the root module's job.

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| `terraform-aws-modules/vpc/aws` | Genuinely used in production at many companies — don't reinvent the VPC |
| Custom module with object inputs | Exactly the structure of my `terraform-fullstack` VPC/ECS/ALB modules |
| Git source with `?ref=v1.2.0` | The most common private-module distribution method |
| `for_each` on modules | Stamping out per-environment or per-tenant stacks |
| Module versioning | A module change must not break existing consumers — semver and pinning are how |

---

## Interview Q&A

**Q: What is a Terraform module and why use them?**
> A module is just a directory of .tf files — the directory you run apply in is already the root module, and anything it calls is a child module. The reason to use them is to stop copy-pasting resource blocks between environments. Instead of three near-identical directories for dev, staging, and prod, there's one module and three thin callers passing different inputs. The other benefit is that a module is a contract: it exposes a defined set of inputs and outputs, which means the team consuming it doesn't have to understand the twenty resources inside it.

**Q: How do you design a good module interface?**
> Group related inputs into object variables rather than exposing a long list of flat strings — so a subnet takes a single object with CIDR, availability zone, and a public flag, not three separate variables that can drift apart. Use `optional()` with defaults so callers only specify what they actually care about. Validate inputs so a bad CIDR fails at plan time with a readable message rather than mid-apply with an opaque AWS error. And think about the output shape as part of the interface — I'd return a map keyed by subnet name containing the ID and AZ, rather than a bare list of IDs, because a list forces the caller to depend on positional ordering, which breaks the moment someone inserts a subnet.

**Q: Should a module contain a provider block?**
> No. Modules should declare what providers they need in `required_providers`, but configuring the provider — region, credentials, aliases — is the root module's job, and child modules inherit that configuration. Putting a provider block inside a module makes it impossible for a caller to use that module twice with different regions, and it also blocks the module from being removed cleanly, because Terraform needs the provider configuration to destroy resources. If a module genuinely needs a second provider, the right pattern is to declare a `configuration_aliases` entry and have the caller pass the configured provider in.

**Q: How do you version and distribute modules internally?**
> Either a private registry through Terraform Cloud, or plain Git sources with a tag pin — `git::https://...//modules/vpc?ref=v1.2.0`. The critical part either way is pinning to a tag rather than tracking a branch, because a branch means an upstream commit can change your infrastructure on the next init. We'd version with semver, and treat a breaking change to the module's input or output interface as a major bump. For the module's own provider constraints I'd keep them loose, using `>=` rather than an exact pin, because a module that pins an exact provider version can't be combined with any other module that pins differently.

**Q: What's the difference between `count` on a module versus on a resource?**
> Mechanically it's identical — both meta-arguments work on module blocks the same way they work on resources. `for_each` on a module is how you stamp out repeated stacks: one module block iterating over a map of environments gives you `module.app["dev"]` and `module.app["prod"]` from a single definition. The same count-versus-for_each reasoning applies, and it applies harder, because a module instance contains many resources — an index shift on a module recreates everything inside it, not just one object.

---

## Gotchas & flags

- Changing a module's `source` or `version` requires `terraform init` again.
- Registry module updates can change infrastructure. **Always pin `version`.**
- Module outputs must be **explicitly declared** — a child module's resource attributes are not accessible from the parent unless there's an `output` block for them.
- `for_each` on a module: an index shift recreates every resource inside the module instance. Higher stakes than on a single resource.
- `terraform-<PROVIDER>-<NAME>` repo naming is **enforced** by the registry, not a suggestion.
- Local module changes take effect immediately (no re-init needed for `./path` sources), but changing the source *path* does require re-init.

---

Next: [[11-Dependencies-Lifecycle-Validations]]
