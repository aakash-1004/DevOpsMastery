---
title: "09 — State Manipulation & Import"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - state
  - import
  - brownfield
  - cli
up: "[[Terraform-Mastery-MOC]]"
prev: "[[08-State-Management-and-Remote-Backend]]"
next: "[[10-Modules]]"
origin: Terraform-Mastery/09-State-Manipulation-and-Import.md
---

# 09 — State Manipulation & Import

← Back to [[Terraform-Mastery-MOC]]

> **Brownfield migration — "we have 200 resources built by hand, how do you bring them under Terraform?" — is a standard senior-level interview scenario.** This note is the answer.

---

## State CLI commands

```bash
terraform state list                                  # all resource addresses in state
terraform state show aws_instance.myserver            # full attributes of one resource
terraform state mv <source> <destination>             # rename / relocate a state entry
terraform state rm aws_instance.myserver              # forget a resource (does NOT delete it)
terraform state pull > backup.tfstate                 # download remote state to stdout
terraform state push backup.tfstate                   # upload a local state file to the backend
terraform state replace-provider <from> <to>          # swap provider source addresses
terraform state                                       # list all subcommands
```

### `state list`
First thing to run when you inherit an unfamiliar config. Shows every managed address including indexed and keyed instances:
```
aws_vpc.my-vpc
aws_subnet.main[0]
aws_subnet.main[1]
aws_instance.main["ubuntu"]
module.vpc.aws_internet_gateway.main[0]
```
Filter with a partial address: `terraform state list module.vpc`

### `state show`
Full attribute dump for one resource. Used to find the real cloud ID, or to verify what an import actually pulled in.

### `state mv` — the most useful and most dangerous
Renames a state entry **without touching real infrastructure**. Terraform treats a renamed resource as a delete + create unless you move the state entry first.

```bash
# Renamed the resource in your config
terraform state mv aws_instance.old_name aws_instance.new_name

# Moved a resource into a module
terraform state mv aws_vpc.main module.network.aws_vpc.main

# Migrating count → for_each (the classic)
terraform state mv 'aws_subnet.main[0]' 'aws_subnet.main["public"]'
terraform state mv 'aws_subnet.main[1]' 'aws_subnet.main["private"]'
```

That last case is the practical fix for the count-shifting problem → [[07-Count-vs-ForEach]].

**Modern alternative — `moved` blocks (Terraform 1.1+).** Declarative, version-controlled, and safe for consumers of a module:
```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```
Applied automatically on the next plan, no CLI gymnastics. **Prefer this over `state mv` for anything in a shared module** — module consumers get the refactor for free instead of being told to run manual commands. Knowing `moved` blocks exist is a strong signal in an interview.

### `state rm` — forget without destroying
```bash
terraform state rm aws_instance.myserver
```
Terraform stops managing the resource; **the resource keeps running in AWS**. Use when:
- Handing a resource over to another team's Terraform config
- Removing a resource from Terraform's control deliberately
- Cleaning up a botched import

The inverse of `import`. **Warning:** now it's an orphan — it exists, it costs money, and no Terraform config tracks it. Document it or you've just created shadow infrastructure.

### `state pull` / `push`
```bash
terraform state pull > backup-$(date +%F).tfstate   # back up before anything risky
terraform state push fixed.tfstate                  # last-resort recovery
```
**Always `state pull` before any state surgery.** It's the undo button.

---

## `terraform import`

Brings **existing** infrastructure under Terraform management.

**The process:**

```hcl
# 1. Write an empty (or partial) resource block matching what you want to adopt
resource "aws_instance" "main" {
}
```
```bash
# 2. Import, giving the resource address and the real cloud ID
terraform import aws_instance.main i-0abc123def456

# 3. Inspect what came in
terraform state show aws_instance.main
```
```hcl
# 4. Fill the config in to match, using what state show reported
resource "aws_instance" "main" {
  ami                    = "ami-0c0e147c706360bd7"
  instance_type          = "t3.nano"
  subnet_id              = "subnet-0abc..."
  vpc_security_group_ids = ["sg-0abc..."]
  tags = { Name = "ExistingServer" }
}
```
```bash
# 5. Plan until it reports no changes — that's the definition of done
terraform plan    # target: "No changes. Your infrastructure matches the configuration."
```

**Step 5 is the whole point.** An import isn't finished when the command succeeds — it's finished when plan is clean. If plan wants to change something, your config doesn't match reality yet, and applying would modify live infrastructure.

**Critical limitation:** `terraform import` imports into **state only**. It does not generate configuration. You write the HCL by hand.

---

## `import` blocks — the modern way (Terraform 1.5+)

```hcl
import {
  to = aws_instance.main
  id = "i-0abc123def456"
}

resource "aws_instance" "main" {
  # ... config
}
```

Advantages over the CLI command:
- **Shows up in `terraform plan`** — you see what will be imported before committing
- Version-controlled and reviewable in a PR
- Can be applied in bulk rather than one command per resource

And config generation:
```bash
terraform plan -generate-config-out=generated.tf
```
Terraform writes HCL matching the imported resources. Output needs cleanup — it dumps every attribute including read-only ones — but it beats writing a hundred resource blocks by hand.

**This is the answer to "how would you migrate 200 manually-created resources?"** — import blocks plus generated config, done in batches by service, not 200 CLI invocations.

---

## Finding import IDs

Import ID format differs per resource type and is documented on each resource's registry page. Common ones:

| Resource | Import ID |
|---|---|
| `aws_instance` | `i-0abc123` |
| `aws_s3_bucket` | bucket name |
| `aws_vpc` | `vpc-0abc123` |
| `aws_subnet` | `subnet-0abc123` |
| `aws_security_group` | `sg-0abc123` |
| `aws_iam_user` | user name |
| `aws_iam_role_policy_attachment` | `role-name/policy-arn` — composite |
| `aws_route_table_association` | `subnet-id/rtb-id` — composite |

**Composite IDs are the trap.** People assume every import ID is a single ARN or ID. Always check the registry docs for the resource.

Tools that generate config + imports from live infrastructure: **Terraformer** (Google), **Former2** (CloudFormation-oriented but useful). Worth naming in an interview to show awareness — but flag that output always needs review.

---

## Brownfield migration strategy

The full answer to the scenario question:

1. **Inventory** — tag or list everything that needs adopting; decide scope
2. **Start with the stable, low-risk layer** — VPC, subnets, security groups. Networking changes rarely, so a mistake is less likely to cause an outage.
3. **Write config to match reality exactly** — do not "improve" things during the import. Refactor later, in a separate change.
4. **Import in batches**, one logical group at a time
5. **`terraform plan` must be clean** before moving to the next batch. Non-negotiable.
6. **Only after everything is imported and stable**, start making changes
7. **Lock down console write access** so drift doesn't immediately reintroduce the problem

**The single most important discipline:** never mix importing with changing. Import until plan is clean, then change in a separate PR. Mixing them means a plan that wants to modify things and you can't tell whether that's your intent or a config mismatch.

---

## Real-world mapping

| Command | When you actually reach for it |
|---|---|
| `state list` | First thing on inheriting an unfamiliar repo |
| `state show` | Debugging why plan wants to change something unexpectedly |
| `state mv` / `moved` blocks | Refactoring resources into modules, count → for_each migration |
| `state rm` | Handing ownership of a resource to another team |
| `state pull` | Backup before any state surgery — always |
| `import` | Brownfield adoption; also recovering after state loss |
| `force-unlock` | A CI job was killed mid-apply and left the lock held |

---

## Interview Q&A

**Q: You've joined a team with 200 AWS resources built by hand. How do you bring them into Terraform?**
> Incrementally, and starting with the stable layer. I'd begin with networking — VPC, subnets, security groups — because that changes rarely so mistakes are lower risk, and everything else depends on it anyway. For each resource I write a config block matching what actually exists, then use import blocks with `-generate-config-out` to bootstrap the HCL rather than hand-writing two hundred blocks. The rule I hold to is that plan must come back clean before I move to the next batch — an import isn't done when the command succeeds, it's done when Terraform reports no changes, because a dirty plan means my config doesn't match reality and applying would modify live infrastructure. And critically, I don't improve anything during migration. Adopt first, refactor later, in separate changes.

**Q: What's the difference between `terraform import` and a data source?**
> Import takes ownership — the resource goes into state, Terraform manages it from then on, and a destroy will delete it. A data source is read-only; Terraform looks the resource up and uses its attributes but never modifies or deletes it. So the question is really about intent: if another team owns the VPC and I just need its ID, that's a data source. If my team is taking over management of that VPC, that's an import. Getting this wrong in the import direction is the dangerous one, because you can end up destroying infrastructure that belongs to someone else.

**Q: You renamed a resource in your config and now plan wants to destroy and recreate it. How do you avoid that?**
> Terraform identifies resources by their address, so a rename looks like one resource disappearing and a different one appearing. The fix is to move the state entry rather than let it recreate — either `terraform state mv old.address new.address`, or better, a `moved` block in the configuration, which is declarative and gets applied automatically on the next plan. I prefer `moved` blocks especially inside modules, because they're version-controlled and reviewable, and module consumers get the refactor automatically instead of being told to run manual CLI commands.

**Q: What does `terraform state rm` do — does it delete the resource?**
> No, and that's the whole point of it. It removes the entry from state so Terraform stops tracking the resource, but the resource keeps running in AWS. I'd use it when handing a resource over to another team's configuration, or to clean up after a botched import. The thing to be careful about is that the resource is now an orphan — it exists and costs money but no Terraform configuration tracks it — so it has to be documented or imported somewhere else immediately, otherwise you've just created shadow infrastructure.

**Q: Before doing anything risky with state, what do you do?**
> `terraform state pull` to a timestamped file. That's the backup and the undo button. State surgery is one of the few Terraform operations with no built-in safety net, so having the previous state on disk means a bad `state mv` or `state rm` is recoverable with a `state push`. On top of that, versioning on the S3 state bucket gives a second layer, which is why I'd never run a state bucket without it.

---

## Gotchas & flags

- `terraform import` **only writes state.** It does not generate configuration. `-generate-config-out` with import blocks is the workaround.
- Import IDs are **resource-type specific** and some are composite (`subnet-id/rtb-id`). Check the registry docs.
- Importing a resource whose config doesn't match reality gives a dirty plan. Applying that plan modifies live infrastructure. Always plan to clean before proceeding.
- `state rm` on a resource you still want means Terraform will try to **create it again** on the next apply, causing a name collision or a duplicate.
- `state mv` between **different state files** requires `-state` and `-state-out` flags and is genuinely fiddly. `moved` blocks don't work across state files either.
- Bulk `state mv` in PowerShell needs careful quoting for indexed addresses — single quotes around `'aws_subnet.main[0]'`.

---

Next: [[10-Modules]]
