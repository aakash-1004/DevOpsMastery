---
title: "08 — State Management & Remote Backend"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - state
  - backend
  - s3
  - dynamodb
  - drift
  - locking
config: tf-backend/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[07-Count-vs-ForEach]]"
next: "[[09-State-Manipulation-and-Import]]"
origin: Terraform-Mastery/08-State-Management-and-Remote-Backend.md
---

# 08 — State Management & Remote Backend

← Back to [[Terraform-Mastery-MOC]]

> **The single highest-value Terraform topic for interviews.** State, locking, and drift come up in almost every DevOps interview at mid-level and above. If only one note gets revised, it's this one.

---

## What state is

`terraform.tfstate` is a JSON file recording **what Terraform has built** — resource addresses, real cloud IDs, every attribute value, and dependency relationships.

**Four jobs it does:**

1. **Mapping** — connects `aws_instance.myserver` in your config to `i-04b09321cc292ba83` in AWS. Without it Terraform has no idea the instance is already yours.
2. **Metadata** — stores dependency ordering, so on `destroy` it can tear down in reverse order even if the resource blocks have been deleted from the config.
3. **Performance** — caches attributes so plan doesn't need to query every resource from scratch.
4. **Collaboration** — when remote, it's the shared source of truth across a team.

> **Memory hook:** state is Terraform's memory. Delete it and Terraform gets amnesia — it will happily try to create everything again, duplicating your entire infrastructure.

---

## The critical mental model

```
CONFIG  (.tf files)        ← what you WANT
   ↕ diffed by plan
STATE   (.tfstate)         ← what Terraform THINKS exists
   ↕ reconciled by refresh
REALITY (actual AWS)       ← what ACTUALLY exists
```

**`plan` diffs config against state.** It refreshes state from reality first, but the diff itself is config vs state. **Drift is when state and reality disagree** — almost always because a human changed something in the console.

---

## Why local state fails in a team

| Problem | Consequence |
|---|---|
| State on one laptop | Nobody else can plan or apply |
| Two people apply at once | Concurrent writes corrupt state or duplicate resources |
| Laptop dies / repo is cloned fresh | State gone → Terraform tries to recreate live infrastructure |
| State committed to Git | **Secrets in plaintext in version control**, plus merge conflicts on a JSON file nobody can hand-merge |

**State contains secrets in plaintext.** RDS passwords, generated IAM passwords, private keys — all of it. That's why `*.tfstate` is in `.gitignore` and why the S3 state bucket needs encryption and tight IAM.

---

## Config built — `tf-backend/`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.54.1"
    }
  }
  backend "s3" {
    bucket = "demo-bucket-9419a72476f569ef"
    key    = "backend.tfstate"
    region = "eu-north-1"
  }
}
```

**This config is incomplete — and the gap is the interview question.** It has no locking, no encryption, no versioning. The production-grade version:

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate-prod"
    key            = "network/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"   # ← state locking
    encrypt        = true                      # ← SSE at rest
    kms_key_id     = "arn:aws:kms:..."         # ← optional, customer-managed key
  }
}
```

Plus, on the bucket itself:
- **Versioning enabled** — recover from a corrupted or accidentally-deleted state file
- **Block public access** — obviously
- **Bucket policy / IAM** restricting who can read it, since it contains secrets

The locking table:
```hcl
resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"            # must be exactly this name

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

> **Note:** newer AWS provider versions support `use_lockfile = true` for S3-native conditional-write locking, reducing the need for DynamoDB. DynamoDB locking is still what the overwhelming majority of existing infrastructure uses, so know it as the default answer and mention the newer option as awareness.

---

## The chicken-and-egg problem

The S3 bucket and DynamoDB table that hold your state can't themselves be created by a config that uses them as a backend. Two ways out:

1. **Bootstrap config** — a small separate Terraform config with local state that creates the bucket and table. Its own state is either committed (it's tiny and low-risk) or accepted as disposable, since the bucket and table are easy to recreate.
2. **Create them manually / via CLI once**, then reference them.

Then, on the main config:
```bash
terraform init      # detects the new backend block
# "Do you want to copy existing state to the new backend?" → yes
```
Terraform migrates local state into S3.

**`backend` blocks cannot use variables or interpolation.** Everything must be a literal. For per-environment backends you use partial configuration:
```hcl
terraform {
  backend "s3" {}    # empty — filled at init time
}
```
```bash
terraform init -backend-config="key=prod/terraform.tfstate" \
               -backend-config="bucket=mycompany-tfstate-prod"
```
or `-backend-config=prod.backend.hcl`. **"Why can't you use variables in a backend block?"** is a real interview question — the answer is that the backend is initialised before variables are evaluated, so there's nothing to interpolate from yet.

---

## State locking

When `apply` starts, Terraform writes a lock entry to DynamoDB keyed on `LockID`. Any other apply against the same state fails immediately with "Error acquiring the state lock" and shows who holds it, from which host, and since when.

**Without locking:** two engineers apply simultaneously, both read the same state, both write — the second overwrites the first. You get orphaned resources that exist in AWS but not in state, and nobody notices until the bill arrives or a destroy leaves things behind.

**Stuck lock** (someone's CI job was killed mid-apply):
```bash
terraform force-unlock <LOCK_ID>
```
**Dangerous** — only after confirming no apply is actually running. The lock ID is printed in the error message. In an interview, always add that caveat; reaching for `force-unlock` reflexively is a red flag.

---

## Backend types

| Backend | Locking | Notes |
|---|---|---|
| `local` | file lock only | default, single-user |
| `s3` | DynamoDB (or native lockfile) | **the AWS standard** |
| `azurerm` | blob lease | Azure equivalent |
| `gcs` | native | GCP equivalent |
| `remote` / `cloud` | native | Terraform Cloud/Enterprise |
| `consul`, `http`, `pg` | varies | niche |

---

## Drift — what it is and how to handle it

**Definition:** the real infrastructure no longer matches what state says. Cause is almost always a manual change — someone SSH'd into the console during an incident and changed a security group rule, resized an instance, or added a tag.

**Detect:**
```bash
terraform plan                      # refresh happens automatically; any diff = drift
terraform plan -detailed-exitcode   # exit 0 = no changes, 1 = error, 2 = drift
terraform plan -refresh-only        # show ONLY drift, no config-driven changes
```
`-detailed-exitcode` is what a scheduled CI drift-detection job keys off. That's the concrete answer to "how do you detect drift proactively" — a nightly pipeline running plan with detailed exit codes, alerting on 2.

**Resolve — three options, and the right answer depends on intent:**

1. **Revert the drift** — just run `terraform apply`. Terraform pushes reality back to match config. Correct when the manual change was unauthorised.
2. **Adopt the drift** — update the `.tf` config to match what's now in AWS, then apply (a no-op). Correct when the manual change was a legitimate emergency fix that should be kept.
3. **Accept it permanently** — `lifecycle { ignore_changes = [...] }` for attributes that are legitimately managed outside Terraform, like a tag written by an autoscaler or a desired_count adjusted by autoscaling → [[11-Dependencies-Lifecycle-Validations]]

```bash
terraform apply -refresh-only   # update state to match reality WITHOUT changing infra
```
This is the modern replacement for the deprecated `terraform refresh`.

**Prevention — the answer that actually impresses:**
- Remove console write access in production; humans get read-only, Terraform's role does the writing
- Break-glass IAM role for emergencies, with CloudTrail alerting on its use
- Scheduled drift detection in CI
- All changes through PRs

> **Interview framing:** drift is a *process* problem as much as a technical one. Answering only "run apply to fix it" is a mid-level answer. Adding "and then I'd look at why someone had console write access in prod" is a senior answer.

---

## State file structure

```json
{
  "version": 4,
  "terraform_version": "1.9.0",
  "serial": 12,
  "lineage": "8f4a...",
  "outputs": { ... },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "myserver",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        { "attributes": { "id": "i-04b09...", "public_ip": "13.233.126.34", ... } }
      ]
    }
  ]
}
```

- `serial` increments on every write — used to detect stale writes
- `lineage` is a unique ID for this state's history — prevents accidentally pushing an unrelated state file over it
- `mode: "managed"` = resource; `mode: "data"` = data source

**Never hand-edit state.** Use `terraform state` commands → [[09-State-Manipulation-and-Import]]

---

## Splitting state

One giant state file is an anti-pattern at scale:
- Plan gets slow (every resource refreshed every time)
- Blast radius is everything
- One lock blocks the whole org

**Split by lifecycle and ownership** — network, data, application layers each with their own state file and their own backend key. Wire them together with `terraform_remote_state` data sources → [[06-Data-Sources]].

```
s3://mycompany-tfstate/
  network/terraform.tfstate       ← changes rarely, networking team
  data/terraform.tfstate          ← RDS, changes rarely
  app-prod/terraform.tfstate      ← changes daily, app team
  app-staging/terraform.tfstate
```

> **Rule of thumb:** things that change at different rates, or are owned by different teams, belong in different state files.

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| S3 + DynamoDB backend | The default on every AWS shop. This is what `terraform-fullstack` already does. |
| State locking | Prevents two CI jobs racing. Comes up the first time two PRs merge close together. |
| Drift detection | Nightly pipeline running `plan -detailed-exitcode`, alerting on exit 2 |
| `terraform_remote_state` | Network team publishes outputs; app teams consume them read-only |
| State splitting | Mandatory past ~100 resources, or as soon as more than one team is involved |

**Connecting to JS7/Keycloak support experience:** drift is structurally identical to a production incident where the config file says one thing and the running process is doing another. The diagnostic instinct — compare declared vs actual, find who changed what, decide whether to revert or adopt — transfers directly. That's a genuine bridge to use in interviews rather than pretending to more Terraform-production time than I have.

---

## Interview Q&A

**Q: What is the Terraform state file and why does it exist?**
> It's a JSON file that maps the resources in my configuration to the real objects in the cloud — so `aws_instance.web` in my code maps to a specific instance ID in AWS. It exists because Terraform needs to know what it already created; without it, every apply would try to create everything again. It also stores dependency metadata, which is how destroy knows what order to tear things down in, and it caches attributes so plan doesn't have to query everything from scratch. The important thing to know is that it contains every attribute value including secrets in plaintext, so it's sensitive data and never goes in Git.

**Q: How do you manage state for a team?**
> Remote backend. On AWS that's S3 for the state file with a DynamoDB table for locking. S3 gives a shared source of truth with versioning so I can roll back a corrupted state, and server-side encryption because the file contains secrets. DynamoDB gives mutual exclusion — when someone starts an apply, Terraform writes a lock entry, and any concurrent apply fails immediately rather than racing. Without locking, two simultaneous applies can each read the same state and the second write silently overwrites the first, leaving resources that exist in AWS but not in state.

**Q: What is drift and how do you deal with it?**
> Drift is when the real infrastructure stops matching what the state file says, and it's nearly always because someone made a manual change in the console. I detect it by running `terraform plan`, which refreshes state from the cloud first — any unexpected diff is drift. For proactive detection I'd run a scheduled pipeline using `plan -detailed-exitcode`, which returns exit code 2 specifically when there are changes, and alert on that. Resolving it depends on intent: if the manual change was unauthorised, apply reverts it. If it was a legitimate emergency fix, I update the configuration to match and then apply as a no-op so the code is the source of truth again. If it's an attribute genuinely managed outside Terraform, like a count adjusted by autoscaling, I add `ignore_changes` for that attribute. But the real fix is process — production console access should be read-only, with a break-glass role that triggers an alert when used.

**Q: Someone deleted the state file. What now?**
> First, check whether the bucket has versioning enabled — if it does, this is a restore operation and the incident is over in two minutes. That's exactly why versioning on the state bucket is non-negotiable. If there's no version to restore, the infrastructure still exists in AWS but Terraform no longer knows about it, so a plan would try to create everything again. The recovery is importing each resource back into state with `terraform import`, or `import` blocks in newer versions, which is slow and error-prone for a large estate. Depending on how big it is, rebuilding into a fresh environment and cutting over can genuinely be faster than importing hundreds of resources by hand.

**Q: Why can't you use variables in a backend block?**
> Because the backend is initialised before Terraform evaluates variables — at `init` time there's nothing to interpolate from yet. The way around it is partial configuration: leave the backend block empty or minimal in code, and supply the rest at init with `-backend-config` flags or a backend config file per environment. That's how you get one configuration pointing at different state keys for dev, staging, and prod.

**Q: How do you split state across a large estate?**
> By lifecycle and ownership. Things that change at different rates or belong to different teams get separate state files — so networking in one, databases in another, each application in its own, with separate backend keys. The benefits are that plan stays fast because each state only refreshes its own resources, blast radius is contained, and one team's lock doesn't block everyone else. The stacks are wired together with `terraform_remote_state` data sources, where the network stack publishes outputs and the application stacks consume them read-only, so there's an explicit contract between them.

---

## Gotchas & flags

- **`.gitignore` must exclude `*.tfstate` and `*.tfstate.*`.** The course's gitignore does this correctly.
- **The course's `.gitignore` also excludes `.terraform.lock.hcl` — that's wrong.** The lock file **should be committed**; it's what guarantees everyone resolves identical provider versions and hashes. Excluding it defeats the purpose of having it.
- `terraform refresh` is deprecated. Use `terraform apply -refresh-only`.
- State bucket versioning is the difference between a two-minute restore and a multi-day import exercise.
- `force-unlock` is a last resort, after confirming no apply is running.
- Backend blocks take literals only — no variables, no interpolation.
- `-target` exists for surgical operations but is explicitly documented as an exceptional tool. Routine use of `-target` means the config should have been split instead.

---

Next: [[09-State-Manipulation-and-Import]]
