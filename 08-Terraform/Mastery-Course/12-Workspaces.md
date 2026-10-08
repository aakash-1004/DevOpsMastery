---
title: 12 — Workspaces
type: concept
module: terraform
status: complete
interview-relevance: ★★★☆☆
tags:
  - terraform
  - workspaces
  - multi-environment
config: tf-cli-workspace/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[11-Dependencies-Lifecycle-Validations]]"
next: "[[13-AWS-Networking-VPC-Concepts]]"
origin: Terraform-Mastery/12-Workspaces.md
---

# 12 — Workspaces

← Back to [[Terraform-Mastery-MOC]]

> **Interview framing matters more than mechanics here.** The commands are trivial. What's tested is whether you know *when not to use workspaces* — and most experienced engineers don't use them for environments.

---

## Concept

A workspace lets one configuration directory hold **multiple independent state files**. Same code, different state, different resources.

```
        tfstate          tfstate           tfstate
           │                │                 │
    workspace-dev    workspace-test    workspace-prod
           └────────────────┴─────────────────┘
                            │
                       one tf config
```

Every configuration starts in a workspace called `default`. You've been using workspaces the whole time without knowing it.

---

## Commands

```bash
terraform workspace list                # show all; * marks the current one
terraform workspace new dev             # create AND switch to it
terraform workspace select prod         # switch
terraform workspace show                # current workspace name
terraform workspace delete dev          # must switch away first, and it must be empty
```

Deleting:
```bash
terraform workspace select default
terraform workspace delete dev          # fails if dev's state still has resources
```

---

## Config built — `tf-cli-workspace/`

```hcl
resource "random_id" "rand_id" {
  byte_length = 8
}

resource "aws_s3_bucket" "demo-bucket" {
  bucket = "demo-bucket-${terraform.workspace}-${random_id.rand_id.hex}"
}
```

`terraform.workspace` is a built-in reference returning the current workspace name as a string.

Run in three workspaces →
```
demo-bucket-dev-9419a72476f569ef
demo-bucket-test-3f8b21c4e7a9d012
demo-bucket-prod-c1d4e8f2a6b39745
```

**Interpolating `terraform.workspace` into resource names is mandatory, not optional.** Without it, the second workspace's apply collides with the first workspace's resources on any globally-unique name — S3 buckets, IAM roles, RDS identifiers. This is the #1 workspace mistake.

Common pattern for varying config per workspace:
```hcl
locals {
  instance_type = {
    dev  = "t3.micro"
    prod = "m5.large"
  }
}

resource "aws_instance" "app" {
  instance_type = local.instance_type[terraform.workspace]
}
```

---

## Where state goes

| Backend | Default workspace | Named workspace |
|---|---|---|
| local | `terraform.tfstate` | `terraform.tfstate.d/<name>/terraform.tfstate` |
| s3 | `<key>` | `env:/<workspace>/<key>` |

**This is the crux of the criticism.** All workspaces share **one backend configuration** — one bucket, one DynamoDB table, one set of credentials. Prod state sits in the same bucket as dev state, differing only by key prefix.

---

## Why experienced teams avoid workspaces for environments

| Problem | Consequence |
|---|---|
| One backend for all environments | Can't put prod state in a separate, locked-down AWS account |
| Environment is **implicit CLI state** | `terraform apply` after forgetting to `workspace select` applies to whatever workspace you were last in. **Nothing in the code tells you which environment you're targeting.** |
| Same code for every environment | Prod usually needs things dev doesn't — multi-AZ, read replicas, WAF. Expressing that means conditionals scattered through the config. |
| Same IAM identity | The credentials that can touch dev can touch prod |
| No PR-visible difference | A code review can't tell you which environments a change affects |

**The `-var-file` alternative — the answer to give:**

```
environments/
├── dev/
│   ├── main.tf            # or a thin module call
│   ├── backend.hcl        # dev bucket / dev key
│   └── terraform.tfvars
├── prod/
│   ├── main.tf
│   ├── backend.hcl        # prod bucket, possibly a different AWS account
│   └── terraform.tfvars
└── modules/
    └── app/               # the shared code lives here
```

Each environment has its own directory, own backend, own state, own credentials. The shared logic lives in modules so there's still no duplication of the actual resource code. **The environment is explicit in the path you `cd` into** — you can't apply to prod by accident.

---

## Where workspaces genuinely fit

- **Ephemeral, short-lived environments** — a per-PR preview environment named after the branch, created and destroyed by CI
- **Testing a change in isolation** before applying to the real state
- **Small, single-team, low-risk setups** where the operational overhead of separate directories isn't justified
- Cases where the environments really are structurally identical

> **The honest position:** workspaces are a legitimate feature that gets misapplied. For genuinely identical, short-lived environments they're the right tool. For dev/staging/prod, separate directories with separate backends win on blast radius and explicitness.

---

## CLI workspaces vs Terraform Cloud workspaces

**These are different concepts with the same name** — a genuine source of confusion worth being precise about.

| | CLI workspace | Terraform Cloud workspace |
|---|---|---|
| Is | An alternate state file in one backend | A full managed environment |
| Has own variables? | No | **Yes** |
| Has own credentials? | No | **Yes** |
| Has own VCS connection? | No | **Yes** |
| Has run history / approvals? | No | **Yes** |
| Suitable for prod isolation? | Weakly | **Yes** |

TFC workspaces solve exactly the problems CLI workspaces don't — per-workspace variables, per-workspace credentials, per-workspace access control. → [[17-Terraform-Cloud-and-Best-Practices]]

---

## Interview Q&A

**Q: What are Terraform workspaces?**
> They let one configuration directory maintain multiple independent state files, so the same code can manage several sets of resources. Every config starts in a workspace called `default`, and you create more with `terraform workspace new`. Inside the configuration, `terraform.workspace` gives you the current workspace name, which you interpolate into resource names so the environments don't collide on globally-unique names like S3 buckets.

**Q: Would you use workspaces for dev, staging, and production?**
> Generally no, and the reason is that all workspaces share one backend configuration — one bucket, one lock table, one set of credentials. That means production state lives alongside dev state and I can't isolate it into a separate AWS account, which is the isolation most organisations actually want. The bigger problem is that the target environment is implicit CLI state rather than something in the code: if I forget to run `workspace select`, apply goes to whichever workspace I was last in, and nothing in the repository or a pull request tells a reviewer which environments a change affects. I'd use separate directories per environment, each with its own backend config and tfvars, sharing the actual resource code through modules. The environment becomes the directory you're standing in, which you can't get wrong silently.

**Q: So when *are* workspaces the right tool?**
> Short-lived, genuinely identical environments. A per-pull-request preview environment created and destroyed by CI is the ideal case — the workspace name comes from the branch, it exists for hours, and there's no need for separate credentials because it's all in the dev account anyway. Also useful for testing a risky change against a throwaway state before touching the real one. The common thread is that the environments are structurally the same and nothing valuable lives in them.

**Q: Are Terraform Cloud workspaces the same thing?**
> No, and the shared name causes real confusion. A CLI workspace is just an alternate state file within one backend. A Terraform Cloud workspace is a full managed environment — its own variables, its own credentials, its own VCS connection, its own run history and approval gates, and its own access controls. TFC workspaces actually solve the isolation problem that CLI workspaces don't, which is why "use workspaces for environments" is reasonable advice in Terraform Cloud and poor advice for the CLI.

---

## Gotchas & flags

- **Forgetting `workspace select` is the classic incident.** Always `terraform workspace show` before an apply, and put the workspace name in your shell prompt if you use them seriously.
- Not interpolating `terraform.workspace` into globally-unique resource names causes collisions the moment you use a second workspace.
- `workspace delete` fails if the workspace's state still contains resources — destroy first.
- The `default` workspace cannot be deleted.
- `terraform.workspace` in a module returns the **root module's** workspace, not something module-specific.
- Workspaces do **not** isolate credentials. Anyone who can apply to dev can apply to prod.

---

Next: [[13-AWS-Networking-VPC-Concepts]]
