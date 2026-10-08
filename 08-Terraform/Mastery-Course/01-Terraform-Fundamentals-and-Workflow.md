---
title: "01 — Terraform Fundamentals & Core Workflow"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - iac
  - fundamentals
  - workflow
up: "[[Terraform-Mastery-MOC]]"
prev: "[[Terraform-Mastery-MOC]]"
next: "[[02-HCL-Block-Types]]"
origin: Terraform-Mastery/01-Terraform-Fundamentals-and-Workflow.md
---

# 01 — Terraform Fundamentals & Core Workflow

← Back to [[Terraform-Mastery-MOC]]

---

## Concept

Terraform is an **open-source Infrastructure as Code (IaC)** tool. Instead of clicking through the AWS console, you describe the desired infrastructure in config files, and Terraform figures out the API calls needed to make reality match.

**IaC in one line:** manage infrastructure through version-controlled configuration files rather than a GUI.

**Why it matters over manual console work:**

| Manual console | Terraform |
|---|---|
| Clicks are unrepeatable | Config is repeatable, byte-identical every run |
| No record of *why* something exists | Git history + comments |
| Dev/staging/prod drift apart | Same config, different tfvars |
| Onboarding = "watch me click" | Onboarding = read the repo |
| Rollback = remember what you changed | Rollback = `git revert` + apply |

---

## Terraform config characteristics

- **Extension:** `.tf`
- **Language:** HCL (HashiCorp Configuration Language). JSON (`.tf.json`) is also supported but nobody uses it by hand — it exists for machine generation.
- **Declarative, not imperative.** You declare the *end state*; Terraform computes the *path* to get there. You never write "create this, then create that" — dependency order is derived automatically.
- **State-driven.** Terraform keeps a record of what it has built in `terraform.tfstate`.

> **Declarative vs imperative memory trick:** Imperative = giving turn-by-turn driving directions. Declarative = giving the destination address and letting the GPS route it. Terraform is the GPS.

---

## The core workflow

```bash
terraform init       # download providers, set up backend, create .terraform/
terraform validate   # syntax + internal consistency check — no AWS calls
terraform fmt        # auto-format to canonical HCL style
terraform plan       # diff config vs state — shows what WILL change, changes nothing
terraform apply      # execute the diff against the real cloud, then update state
terraform destroy    # tear down everything in state
```

### What each command actually does

**`terraform init`**
- Reads `required_providers`, downloads the provider plugin binaries into `.terraform/providers/`
- Initialises the backend (local file, or S3 if a `backend` block exists)
- Writes `.terraform.lock.hcl` — the dependency lock file recording exact provider versions + hashes
- **Run it:** first time in a directory, after adding/changing a provider, after changing backend config
- `-upgrade` flag forces re-resolution of version constraints

**`terraform validate`**
- Checks HCL syntax, required arguments, type correctness, reference validity
- **Makes zero API calls** — works offline, no credentials needed
- Caught a real typo during this course: `provisder "aws"` → *"Blocks of type 'provisder' are not expected here. Did you mean 'provider'?"*
- This is the cheap check you run before wasting a plan cycle

**`terraform plan`**
- Refreshes state (queries AWS for the current attributes of tracked resources)
- Diffs **desired config** against **state**
- Prints an execution plan with symbols: `+` create, `-` destroy, `~` update in place, `-/+` destroy and recreate
- **Read-only against your infrastructure** — nothing is modified
- `-out=tfplan` saves the plan to a file so `apply` executes *exactly* that plan and can't drift between the two steps. **This is what you do in CI/CD.**

**`terraform apply`**
- Runs a plan, asks for confirmation (`yes`), then makes the real API calls
- Writes the resulting reality into `terraform.tfstate`
- `-auto-approve` skips the prompt — fine in CI, dangerous manually
- `apply tfplan` applies a saved plan file, no prompt, no re-plan

**`terraform destroy`**
- Plans and executes deletion of everything in state
- `-target=aws_instance.myEc2` destroys one resource only

---

## The `(known after apply)` marker

In plan output, attributes AWS assigns at creation time show as `(known after apply)` — `id`, `arn`, `public_ip`, `private_dns`, `availability_zone`. Terraform can't know them until the resource exists.

**Interview-relevant point:** this is why you can't always fully validate a plan. If a value is `(known after apply)` and another resource depends on it, downstream changes may also be unknown. Deeply chained unknowns are why plans sometimes look vaguer than expected.

---

## Terraform CLI vs OpenTofu (2025 context)

HashiCorp moved Terraform from MPL to the BUSL licence in Aug 2023. The community fork is **OpenTofu** (now a Linux Foundation project), CLI-compatible with Terraform ~1.5.x.

**If asked in an interview:** most enterprises are still on Terraform proper; OpenTofu adoption is real but smaller. Know that the fork exists and why — it signals you follow the ecosystem, not just the syntax.

---

## Real-world mapping

| Course concept | Where it shows up on the job |
|---|---|
| `plan` before `apply` | PR-gated pipelines post the plan output as a comment; humans approve before apply runs |
| `validate` + `fmt` | Pre-commit hooks and CI lint stages — a PR that fails `terraform fmt -check` doesn't merge |
| `-out=tfplan` | Mandatory in CI: plan runs on PR, saved artifact applies on merge, so nothing changes in between |
| `destroy` | Ephemeral preview environments torn down nightly to control cost |

Ties to my own work: `terraform-fullstack` already uses this loop. What was missing was the **saved-plan discipline** — plan and apply as two separately-approved CI stages.

---

## Interview Q&A

**Q: What is Terraform and why use it over the AWS console or CloudFormation?**
> Terraform is an infrastructure-as-code tool that lets you define cloud resources declaratively in HCL and version them in Git. Over the console, the win is repeatability and auditability — the same config produces the same environment every time, and every change has a Git history. Over CloudFormation, the main advantage is that it's cloud-agnostic, so the same tool and workflow covers AWS, Azure, Kubernetes, and even SaaS providers. It also has a much stronger module ecosystem through the public registry.

**Q: Explain the difference between `terraform plan` and `terraform apply`.**
> Plan is a dry run. It refreshes state, diffs my configuration against that state, and prints exactly what would be created, changed, or destroyed — without touching anything. Apply executes that diff, makes the real API calls, and then writes the new reality back into the state file. In practice I always run plan first, and in a pipeline I save it with `-out` so that apply executes exactly the plan that was reviewed, with no chance of drift between review and execution.

**Q: Why is Terraform called declarative? What does that buy you?**
> I declare the end state I want rather than the steps to get there. Terraform builds a dependency graph from the references between resources and works out the correct order and what can run in parallel by itself. The practical benefit is idempotency — if I run apply twice on an unchanged config, the second run does nothing, because Terraform compares desired state to actual state rather than blindly re-running commands.

**Q: What does `terraform init` actually do?**
> Three things. It downloads the provider plugins listed in `required_providers` into the local `.terraform` directory, it initialises the backend — so if I've configured an S3 backend it connects and checks for existing state there — and it writes the dependency lock file pinning exact provider versions and hashes. I re-run it whenever I add a provider, change a module source, or change backend configuration.

**Q: What's the point of `terraform validate` if `plan` also catches errors?**
> Validate is purely local — syntax, required arguments, type checks, reference correctness — with no API calls and no credentials needed. That makes it fast and it works in a CI lint stage before any cloud auth is set up. Plan catches everything validate does plus provider-level and cloud-level problems, but it costs a round trip to AWS. So validate is the cheap early gate, plan is the expensive real check.

---

## Gotchas & flags

- `terraform validate` passing does **not** mean apply will succeed — it never contacts AWS. An invalid AMI ID for your region passes validate and fails apply.
- `terraform plan` **does** refresh state by default, so it needs valid credentials. `-refresh=false` skips that (faster, but you may plan against stale state).
- `destroy` has no undo. There's no confirmation beyond typing `yes`. Use `prevent_destroy` in a lifecycle block on anything critical → [[11-Dependencies-Lifecycle-Validations]]
- The first `terraform init` for the AWS provider downloads several hundred MB — slow init is normal, not a hang.

---

Next: [[02-HCL-Block-Types]]
