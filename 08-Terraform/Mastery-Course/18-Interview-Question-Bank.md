---
title: 18 — Interview Question Bank
type: interview
module: terraform
status: complete
interview-relevance: ★★★★★
tags:
  - terraform
  - interview
  - question-bank
  - revision
up: "[[Terraform-Mastery-MOC]]"
prev: "[[17-Terraform-Cloud-and-Best-Practices]]"
related:
  - "[[Interview-Prep-MOC]]"
origin: Terraform-Mastery/18-Interview-Question-Bank.md
---

# 18 — Interview Question Bank

← Back to [[Terraform-Mastery-MOC]]

> **Use this as the final pass before an interview.** Answer out loud, then check against the linked note. Reading these silently does nothing — spoken fluency is the gap that matters.

---

## Tier 1 — asked in almost every interview

| # | Question | Note |
|---|---|---|
| 1 | What is Terraform and why use it over the console/CloudFormation? | [[01-Terraform-Fundamentals-and-Workflow]] |
| 2 | `plan` vs `apply`, and why state sits between them | [[01-Terraform-Fundamentals-and-Workflow]] |
| 3 | What is the state file and why does it exist? | [[08-State-Management-and-Remote-Backend]] |
| 4 | How do you manage state for a team? | [[08-State-Management-and-Remote-Backend]] |
| 5 | What is drift, how do you detect it, how do you fix it? | [[08-State-Management-and-Remote-Backend]] |
| 6 | `count` vs `for_each` — which and why? | [[07-Count-vs-ForEach]] |
| 7 | What is a module and how do you design a good one? | [[10-Modules]] |
| 8 | How do you manage dev/staging/prod differences? | [[04-Variables-Deep-Dive]], [[12-Workspaces]] |
| 9 | How do you handle secrets in Terraform? | [[17-Terraform-Cloud-and-Best-Practices]] |
| 10 | How do you bring existing infrastructure under Terraform? | [[09-State-Manipulation-and-Import]] |

---

## Tier 2 — mid-level depth

**Workflow & core**
- What does `terraform init` actually do?
- Why run `validate` if `plan` catches everything?
- What does `-/+` mean in a plan and why does it matter?
- What is `(known after apply)` and why does it appear?
- What's the difference between `terraform destroy` and removing a resource block?

**Variables**
- What's the variable precedence order? *(the env-var-is-low trap)*
- `variable` vs `local` — when do you use each?
- How do you validate input before Terraform touches AWS?
- When would you use an `object` type over separate variables?
- What does `optional()` do?

**State**
- Why can't you use variables in a backend block?
- Someone deleted the state file — what now?
- How do you split state across a large estate?
- What does `terraform state rm` do — does it delete the resource?
- What's `terraform state mv` for, and what's the modern alternative? *(`moved` blocks)*

**Resources & meta-arguments**
- How does Terraform determine creation order?
- When would you use `depends_on`?
- Explain `create_before_destroy` and when it breaks.
- What does `ignore_changes` do and what's the risk?
- Difference between variable validation, precondition, postcondition, and `check`?

**Modules**
- Should a module contain a `provider` block? *(no — and why)*
- How do you version and distribute modules internally?
- What's `for_each` on a module for?
- How do module outputs work?

**Data sources**
- Data source vs `terraform import` — what's the difference?
- How do you avoid hardcoding AMI IDs, and what's the tradeoff?
- How do you consume another team's Terraform outputs? *(`terraform_remote_state`)*

**AWS networking**
- VPC, AZ, subnet — what's the relationship?
- What makes a subnet public?
- Security group vs NACL?
- EC2 in a private subnet can't reach the internet — why?
- How many usable IPs in a /24 in AWS? *(251)*
- When would you use a VPC endpoint over NAT?

**CI/CD & practice**
- How would you run Terraform in a pipeline?
- Why `plan -out` then `apply tfplan`?
- How do you authenticate Terraform in CI without stored keys? *(OIDC)*
- What goes in `.gitignore`, and what must **not**? *(lock file must be committed)*
- Where does Terraform stop and Helm/ArgoCD start?

---

## Tier 3 — scenario questions (the differentiators)

**1. "You run plan and it wants to destroy and recreate 12 resources you didn't touch."**
→ Most likely a `count` index shift after removing a list item, or a changed module source version, or a changed `for_each` key. Check `# forces replacement` annotations to find the triggering attribute. Fix with `state mv` if already applied; migrate to `for_each` to prevent recurrence. → [[07-Count-vs-ForEach]]

**2. "Two engineers applied at the same time and now the state is inconsistent."**
→ Locking wasn't configured. Immediate: `state pull` for a backup, reconcile by comparing state against reality, `import` orphans and `state rm` phantoms. Prevent: DynamoDB locking, and CI as the only apply path so humans never apply from laptops. → [[08-State-Management-and-Remote-Backend]]

**3. "Someone changed a security group in the console during an incident."**
→ Drift. `terraform plan` surfaces it. Decision: was it a legitimate fix? If yes, codify it and apply as a no-op. If no, apply reverts it. Then address the root cause — why did someone have console write access in prod? Break-glass role with CloudTrail alerting, and scheduled drift detection via `plan -detailed-exitcode`. → [[08-State-Management-and-Remote-Backend]]

**4. "We have 200 resources built by hand. Migrate them."**
→ Incrementally, networking first because it's stable. Import blocks with `-generate-config-out` rather than 200 CLI calls. Plan must be clean before the next batch. **Never mix import with improvement** — adopt first, refactor in a separate PR. Then lock down console access. → [[09-State-Manipulation-and-Import]]

**5. "Apply succeeded but the application isn't reachable."**
→ Terraform reports success when the API call succeeds; it doesn't verify the workload. Work outside-in: public IP → route table has IGW route → SG allows the port → NACL allows both directions → service listening on the host → cloud-init logs. Timeout vs connection refused splits the search space in half. → [[15-Project-VPC-EC2-Nginx]], [[13-AWS-Networking-VPC-Concepts]]

**6. "Reduce our Terraform plan time — it takes 8 minutes."**
→ State is too large. Split by lifecycle and ownership into separate state files wired with `terraform_remote_state`. Secondary: reduce unnecessary data sources (each is an API call on every plan), use `-refresh=false` for quick iterations, increase `-parallelism`. Avoid `-target` as a routine fix — needing it repeatedly means the config should have been split.

**7. "How do you prevent someone destroying the production database?"**
→ Layered: `prevent_destroy` lifecycle as an accident guardrail, RDS deletion protection on the resource itself, IAM policy denying delete on prod resources, required PR review, and apply only from CI with an approval gate. Be explicit that `prevent_destroy` alone is not a security control — it's bypassed by removing the resource block. → [[11-Dependencies-Lifecycle-Validations]]

**8. "Your for_each fails with 'depends on resource attributes that cannot be determined until apply'."**
→ The keys derive from a value that doesn't exist at plan time. The reflex answer is a targeted two-stage apply; the real fix is restructuring so keys come from static input — variables, locals, or a config file — rather than resource outputs. → [[07-Count-vs-ForEach]]

**9. "How do you enforce that every resource has a CostCenter tag?"**
→ Provider `default_tags` makes it automatic. Variable validation catches missing inputs. `checkov`/`tfsec` in CI as a gate. Sentinel or OPA at plan time in Terraform Cloud Enterprise for org-wide enforcement. AWS-side backstop: a Config rule or SCP. → [[17-Terraform-Cloud-and-Best-Practices]]

**10. "Terraform is stuck — 'Error acquiring the state lock'."**
→ Someone's apply is running, or a CI job was killed mid-apply and the lock is orphaned. The error prints who holds it, from where, and since when. **Confirm no apply is actually running**, then `terraform force-unlock <ID>`. Reaching for force-unlock without checking first is the wrong instinct. → [[08-State-Management-and-Remote-Backend]]

---

## Rapid-fire — one-line answers

| Question | Answer |
|---|---|
| Is Terraform declarative or imperative? | Declarative — you state the end result, Terraform derives the path |
| Can you write custom functions? | No. Built-ins only. |
| What does `terraform.workspace` return? | Current workspace name as a string |
| What's the default parallelism? | 10 concurrent operations |
| `list` vs `set`? | list ordered + duplicates; set unordered + unique |
| What can `for_each` accept? | A map or a set of strings — not a list |
| Where does `self` work? | Postconditions only |
| What is `.terraform.lock.hcl`? | Provider version + hash lock — **commit it** |
| Does `sensitive = true` encrypt state? | No. Masks output only. |
| Does `state rm` delete the resource? | No — Terraform just stops tracking it |
| Does `import` generate config? | No. `-generate-config-out` with import blocks does. |
| `~> 5.54` vs `~> 5.54.1`? | First allows minor+patch, second allows patch only |
| Usable IPs in an AWS /24? | 251 — AWS reserves 5 |
| Is VPC peering transitive? | No |
| Can a subnet span AZs? | No |
| What makes a subnet public? | A `0.0.0.0/0` route to an IGW in its route table |
| SG stateful or stateless? | Stateful. NACL is stateless. |
| Can backend blocks use variables? | No — literals only, or `-backend-config` at init |
| What replaced `terraform refresh`? | `terraform apply -refresh-only` |
| What does `moved {}` do? | Declarative state relocation for renamed/moved resources |

---

## Honest-gap answers

Things not covered hands-on. **Say so — a fabricated answer collapses under one follow-up.**

**Terraform Cloud:** *"I haven't run Terraform Cloud in production. I've built the equivalent workflow myself with S3 and DynamoDB for state and locking, and GitHub Actions for plan-on-PR and apply-on-merge — so I understand what TFC is replacing. What I'd want to learn hands-on is the Sentinel policy side and how per-workspace variable sets work in practice."*

**Sentinel / OPA:** *"I know what they do — policy enforcement at the plan stage, so a plan violating an organisational rule is blocked before apply. I haven't written policies myself. The closest I've done is variable validation blocks and static scanning, which cover a subset of the same ground at a lower level."*

**Large-scale production Terraform:** *"My Terraform work has been building real infrastructure end to end — VPC, ECS Fargate, ALB, ECR through reusable modules with S3 and DynamoDB remote state — rather than operating a large existing estate. So I'm strong on the authoring side and on the reasoning behind state, locking, and module design, and where I'd expect a learning curve is the operational scale problems: state splitting across dozens of stacks, and coordinating changes across teams."*

> **Rule: be specific about what you *have* done, then name the gap precisely.** A precise gap sounds like self-awareness. A vague one sounds like bluffing.

---

## Talking points from real projects

Weave these in rather than giving generic answers:

- **`terraform-fullstack`** — VPC, ECS/Fargate, ECR, ALB via reusable modules; S3 + DynamoDB remote state with locking. Use for: module design, state management, remote backend.
- **Custom VPC module** — object-typed inputs, `optional()` defaults, CIDR validation with `can(cidrnetmask(...))`, `for_each` over a subnet map, conditional IGW with `count = length(...) > 0 ? 1 : 0`, outputs shaped as maps keyed by name. Use for: module design, count vs for_each, validation.
- **IAM YAML automation** — `yamldecode` + nested for + `flatten` + composite `for_each` keys. Use for: "something non-trivial", many-to-many modelling, and the security critique.
- **`gitops-go-app`** — where Terraform stops and ArgoCD/Helm start. Use for: the infra/workload boundary question.
- **JS7/Keycloak production support** — drift is structurally the same as "config says X, running process does Y". Use for: bridging into troubleshooting scenarios honestly.

---

## Pre-interview checklist

- [ ] `plan` vs `apply` vs state — spoken, out loud, in under 30 seconds
- [ ] Drift: define → detect → three resolution paths → prevent
- [ ] `count` vs `for_each` including the index-shift explanation
- [ ] Remote backend: S3 + DynamoDB + encryption + versioning, and *why each*
- [ ] Module design principles, including the no-provider-block rule
- [ ] Brownfield import strategy, including "plan must be clean"
- [ ] Variable precedence order
- [ ] SG vs NACL
- [ ] What makes a subnet public
- [ ] One non-trivial project walkthrough ready (IAM YAML or custom VPC module)
- [ ] Honest-gap answers rehearsed so they don't sound defensive

---

← [[Terraform-Mastery-MOC]]

<!-- related -->
---
**Related:** [[Interview-Prep-MOC|Interview Prep]]
<!-- /related -->
