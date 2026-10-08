---
title: 04 — Variables Deep Dive
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - variables
  - tfvars
  - validation
  - locals
config: tf-variables/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[03-Providers-and-First-Resource-EC2]]"
next: "[[05-Functions-Expressions-and-Loops]]"
origin: Terraform-Mastery/04-Variables-Deep-Dive.md
---

# 04 — Variables Deep Dive

← Back to [[Terraform-Mastery-MOC]]

---

## Config built — `tf-variables/`

**`variables.tf`**
```hcl
variable "aws_instance_type" {
  description = "What type of instance you want to create?"
  type        = string
  validation {
    condition     = var.aws_instance_type == "t2.micro" || var.aws_instance_type == "t3.micro"
    error_message = "Only t2 and t3 micro allowed"
  }
}

variable "ec2_config" {
  type = object({
    v_size = number
    v_type = string
  })
  default = {
    v_size = 20
    v_type = "gp2"
  }
}

variable "additional_tags" {
  type    = map(string)     # expecting key=value format
  default = {}
}
```

**`terraform.tfvars`**
```hcl
aws_instance_type = "t3.micro"

ec2_config = {
  v_size = 30
  v_type = "gp3"
}

additional_tags = {
  DEPT    = "QA"
  PROJECT = "MYPROJECT_QA"
}
```

**`prod.auto.tfvars`**
```hcl
ec2_config = {
  v_size = 40
  v_type = "gp3"
}
```

**`main.tf`**
```hcl
locals {
  owner = "ABC"
  name  = "MySERVER"
}

resource "aws_instance" "myserver" {
  ami           = "ami-0c0e147c706360bd7"
  instance_type = var.aws_instance_type

  root_block_device {
    delete_on_termination = true
    volume_size           = var.ec2_config.v_size
    volume_type           = var.ec2_config.v_type
  }

  tags = merge(var.additional_tags, {
    Name = local.name
  })
}
```

**What this config demonstrates in one place:** object types, validation, map types, tfvars precedence (`terraform.tfvars` says 30/gp3, `prod.auto.tfvars` says 40/gp3 — **auto.tfvars wins**), locals, and `merge()` to combine a user-supplied tag map with a mandatory tag.

---

## Variable type constraints — all 9

### Primitives
```hcl
variable "name"    { type = string }
variable "count"   { type = number }
variable "enabled" { type = bool }
```

### Collections — many values, same type
```hcl
variable "azs"   { type = list(string) }   # ordered, duplicates allowed, index access: var.azs[0]
variable "ports" { type = set(number) }    # unordered, unique, no index access
variable "tags"  { type = map(string) }    # key/value: var.tags["Name"]
```

**`list` vs `set`:** use `set` when uniqueness matters and order doesn't (port lists, IAM policy ARNs). Use `list` when order matters (AZ-to-subnet placement) or duplicates are legitimate. Practically, `list` dominates; `set` shows up mostly as the input to `for_each`.

### Structural — many values, mixed types
```hcl
variable "ec2_config" {
  type = object({
    v_size = number
    v_type = string
  })
}                                          # named attrs: var.ec2_config.v_size

variable "spec"  { type = tuple([string, number, bool]) }   # positional — rarely used
variable "extra" { type = any }                             # no constraint — escape hatch
```

**`object` is the one that matters.** It's how real modules group related inputs instead of exposing fifteen flat strings. Nested objects and `optional()` defaults make it genuinely powerful:

```hcl
variable "subnet_config" {
  type = map(object({
    cidr_block = string
    az         = string
    public     = optional(bool, false)     # optional with default — TF 1.3+
  }))
}
```

That `optional(bool, false)` pattern is from the custom VPC module → [[10-Modules]]. It's the marker of someone who has actually written modules rather than just consumed them.

**Avoid `any`.** It disables type checking entirely. Seeing `any` all over a repo is a code smell, not flexibility.

> **Mnemonic for the 9:** 3 primitives (one value) → 3 collections (many values, one type) → 3 structural (many values, mixed types).

---

## Variable precedence — highest wins

This is the single most-asked variables question. Memorise the order:

| Priority | Source | Example |
|---|---|---|
| 1 (highest) | `-var` CLI flag | `terraform apply -var="region=ap-south-1"` |
| 2 | `-var-file` CLI flag | `terraform apply -var-file="prod.tfvars"` |
| 3 | `*.auto.tfvars` (alphabetical order among them) | `prod.auto.tfvars` |
| 4 | `terraform.tfvars.json` | — |
| 5 | `terraform.tfvars` | — |
| 6 | Environment variable `TF_VAR_<name>` | `export TF_VAR_region=ap-south-1` |
| 7 (lowest) | `default` in the `variable` block | — |

**The trap:** environment variables are *lower* priority than tfvars files. People assume env vars override everything because that's how most tooling works. In Terraform they don't.

**Proof from the config:** `terraform.tfvars` sets `v_size = 30`, `prod.auto.tfvars` sets `v_size = 40`. Result is **40** — auto.tfvars outranks terraform.tfvars.

> **Memory hook — "CLI beats file beats env beats default."** And within files: explicit `-var-file` > `auto` > plain `terraform.tfvars`.

**Loading rules:**
- `terraform.tfvars` and `*.auto.tfvars` load automatically
- Any other filename (`prod.tfvars`, `dev.tfvars`) needs explicit `-var-file=prod.tfvars`
- A variable with no default and no value supplied → Terraform **prompts interactively**, which hangs a CI pipeline. Always supply values non-interactively in automation.

---

## Validation blocks

```hcl
variable "aws_instance_type" {
  type = string
  validation {
    condition     = var.aws_instance_type == "t2.micro" || var.aws_instance_type == "t3.micro"
    error_message = "Only t2 and t3 micro allowed"
  }
}
```

Better idiom using `contains()`:
```hcl
validation {
  condition     = contains(["t2.micro", "t3.micro", "t3.small"], var.instance_type)
  error_message = "Instance type must be one of: t2.micro, t3.micro, t3.small."
}
```

CIDR validation using `can()` + `cidrnetmask()` (from the custom VPC module):
```hcl
validation {
  condition     = can(cidrnetmask(var.vpc_config.cidr_block))
  error_message = "Invalid CIDR Format - ${var.vpc_config.cidr_block}"
}
```

Validating every element of a map:
```hcl
validation {
  condition     = alltrue([for config in var.subnet_config : can(cidrnetmask(config.cidr_block))])
  error_message = "Invalid CIDR Format"
}
```

**`can()` is the key trick:** it wraps an expression and returns `true`/`false` instead of erroring. So `can(cidrnetmask(x))` answers "is x a valid CIDR?" without blowing up on invalid input.

**Why validation matters:** it fails at **plan time** with a clear message, instead of at apply time with an opaque AWS API error halfway through creating resources. That's the difference between a 5-second feedback loop and a half-created environment you have to clean up.

Multiple `validation` blocks per variable are allowed — all are evaluated.

---

## Sensitive variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

Masks the value in plan/apply output and in any output that references it.

**Critical caveat — this does NOT encrypt state.** The value sits in `terraform.tfstate` in plaintext. `sensitive = true` is display hygiene, not security. Real secret handling:
1. Never commit tfvars containing secrets
2. Encrypt state at rest (S3 with SSE-KMS) and lock down bucket access
3. Better: don't put secrets in Terraform at all — pull them at runtime from Secrets Manager / SSM Parameter Store / Vault via a `data` source, or have the application fetch them itself

> **Personal flag:** given past credential-in-plaintext incidents, this is the rule to internalise — a secret that enters Terraform enters the state file. Design so it never enters.

---

## Locals

```hcl
locals {
  project = "project-01"
  name    = "MySERVER"

  common_tags = {
    Project   = local.project      # locals can reference other locals
    ManagedBy = "terraform"
    Owner     = var.owner          # and variables
  }
}
```

Referenced as `local.x` (**singular**, even though the block is `locals`).

**Use for:** naming prefixes (`"${local.project}-vpc"`), common tag maps, computed filters, anything repeated more than twice.

**`merge()` pattern** — combine caller-supplied tags with mandatory ones:
```hcl
tags = merge(var.additional_tags, {
  Name = local.name
})
```
Later arguments win on key collision, so mandatory tags should come last.

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| tfvars per environment | `dev.tfvars`, `staging.tfvars`, `prod.tfvars` — same config, `-var-file` selects the environment |
| `TF_VAR_` env vars | How CI injects values without writing files — e.g. `TF_VAR_image_tag=$GITHUB_SHA` |
| `validation` blocks | Guardrails: enforce approved instance types, required tags, naming conventions. This is policy-as-code's cheaper cousin. |
| `object` types | Module interfaces. A well-designed module takes 2–3 object variables, not 20 strings. |
| `sensitive = true` | Table stakes, but the real answer is Secrets Manager + a data source |

---

## Interview Q&A

**Q: What's the variable precedence order in Terraform?**
> Highest to lowest: command-line `-var` flags, then `-var-file`, then any `*.auto.tfvars` files in alphabetical order, then `terraform.tfvars`, then `TF_VAR_` environment variables, and finally the default in the variable block. The one that surprises people is that environment variables are near the bottom — a value in terraform.tfvars will silently override a TF_VAR environment variable, which is the opposite of how most tooling behaves.

**Q: How do you manage different values for dev, staging, and production?**
> A tfvars file per environment and `-var-file` to select it — so `terraform apply -var-file=prod.tfvars`. That keeps one set of configuration code with the differences isolated to data. The alternative is separate directories per environment with their own backend keys, which is more verbose but gives stronger blast-radius isolation and separate state files. For anything production-facing I prefer separate directories over workspaces, because workspaces share one backend configuration and it's too easy to apply to the wrong environment.

**Q: How do you validate input before Terraform touches AWS?**
> Validation blocks on the variables. I put a `condition` and an `error_message` in the variable declaration — for example using `contains()` to restrict instance types to an approved list, or `can(cidrnetmask(...))` to check that a CIDR string actually parses. These fail at plan time with a readable message rather than at apply time with an opaque API error, which matters because a mid-apply failure can leave you with half a created environment. For validating every element of a collection I combine `alltrue()` with a for expression.

**Q: What does `sensitive = true` actually protect against?**
> Accidental exposure in CLI output and logs — Terraform masks the value in plan, apply, and output. What it does *not* do is encrypt the state file; the value is still plaintext in terraform.tfstate. So it's display hygiene, not security. The real controls are encrypting state at rest with KMS, restricting access to the state bucket, and ideally keeping secrets out of Terraform entirely by fetching them at runtime from Secrets Manager or SSM Parameter Store.

**Q: When would you use an object type instead of separate variables?**
> When the values are logically one thing and always supplied together. Volume size and volume type belong to the root block device — as two loose variables they can drift apart and the relationship isn't visible in the interface. As an object it's one coherent input. This matters most in module design: a module taking three well-shaped object variables is far easier to consume than one taking twenty flat strings, and with `optional()` I can give individual attributes defaults so callers only specify what they care about.

---

## Gotchas & flags

- A variable with no default and no supplied value **prompts interactively** — this hangs CI. Always pass values non-interactively in automation.
- `variable` defaults must be literal. They cannot reference other variables, resources, or functions. If you need computation, use a `local`.
- `*.auto.tfvars` files load in **alphabetical order**, and later ones override earlier ones. `a.auto.tfvars` loses to `z.auto.tfvars`.
- Committing `terraform.tfvars` is common and usually fine for non-secret values — but **the course's `.gitignore` has the tfvars exclusion commented out**, which is a live footgun. Decide deliberately.
- `local` is singular in references; `locals` is plural only in the block declaration.

---

Next: [[05-Functions-Expressions-and-Loops]]
