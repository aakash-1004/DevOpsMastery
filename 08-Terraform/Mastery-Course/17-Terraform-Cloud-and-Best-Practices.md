---
title: "17 — Terraform Cloud, CI/CD & Best Practices"
type: concept
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - terraform-cloud
  - cicd
  - security
  - best-practices
up: "[[Terraform-Mastery-MOC]]"
prev: "[[16-Project-IAM-YAML-Automation]]"
next: "[[18-Interview-Question-Bank]]"
origin: Terraform-Mastery/17-Terraform-Cloud-and-Best-Practices.md
---

# 17 — Terraform Cloud, CI/CD & Best Practices

← Back to [[Terraform-Mastery-MOC]]

---

## Terraform Cloud

A HashiCorp-managed service for running Terraform collaboratively. Free tier exists for small teams.

**What it provides:**

| Feature | Replaces |
|---|---|
| Remote state management | Building your own S3 + DynamoDB backend |
| VCS integration | Manually wiring plan/apply into a CI pipeline |
| Automated runs | Plan on PR, apply on merge |
| Secure variable management | Secrets in tfvars or CI variables |
| Run history + audit | Reading CI logs |
| Approval gates | Manual coordination |
| Policy as code (Sentinel/OPA) | Homegrown checks |
| Private module registry | Git-source modules with tag pinning |
| Cost estimation | Guesswork |

**The `cloud` block:**
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-network"
    }
  }
}
```
Then `terraform login` authenticates the CLI.

**TFC workspaces ≠ CLI workspaces.** A TFC workspace is a full environment with its own variables, credentials, VCS connection, and access control. That's why it actually solves environment isolation where CLI workspaces don't → [[12-Workspaces]]

**Execution modes:**
- **Remote** — plan and apply run on TFC's infrastructure
- **Local** — TFC stores state only, execution happens on your machine or your CI
- **Agent** — TFC dispatches to a self-hosted agent inside your network (for private infrastructure it can't reach)

**When to use TFC vs your own CI:** TFC if you want the whole workflow managed and don't want to build state infrastructure. Your own CI (GitHub Actions, GitLab CI, Jenkins) if you already have pipelines, want everything in one place, or need to avoid another vendor. Both are common. **Neither answer is wrong in an interview as long as you can articulate the tradeoff.**

---

## Terraform in CI/CD

The standard pipeline shape — this is what gets asked:

```yaml
# Pull request
- terraform fmt -check      # fails the build on unformatted code
- terraform init
- terraform validate
- tfsec / checkov           # security + policy scanning
- terraform plan -out=tfplan
- post plan output as a PR comment

# Merge to main
- terraform apply tfplan    # applies the EXACT reviewed plan
```

**The critical detail: `plan -out=tfplan` then `apply tfplan`.** Applying a saved plan guarantees that what was reviewed is what executes. Running a fresh plan inside apply means the world could have changed between review and execution.

**Authentication in CI — the modern answer:** OIDC federation. GitHub Actions requests a short-lived token from AWS STS by assuming a role, with a trust policy scoped to a specific repo and branch. **No long-lived access keys stored anywhere.**

```yaml
permissions:
  id-token: write
  contents: read

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::956179206096:role/github-actions-terraform
    aws-region: ap-south-1
```

Saying "OIDC federation, not stored access keys" is a current-practice signal and directly relevant given my GitHub Actions experience on `gitops-go-app`.

**Where Terraform stops and Kubernetes tooling starts** — a boundary question that comes up:
- **Terraform** provisions the infrastructure layer: VPC, EKS cluster, node groups, IAM roles, RDS, ALB
- **Helm / ArgoCD** manage the workload layer: deployments, services, ingresses, config

Terraform *can* manage Kubernetes resources via the `kubernetes` and `helm` providers, but it's generally the wrong tool — application deployments change many times a day and belong in a GitOps loop, not in an infrastructure state file. Keeping the boundary clean is the answer. That's exactly the split between my `terraform-fullstack` and `gitops-go-app` repos.

---

## Security

### Secrets
1. **Never** hardcode credentials in `.tf` files
2. **Never** commit tfvars containing secrets
3. `sensitive = true` masks output but **does not encrypt state**
4. Pull secrets at runtime from Secrets Manager / SSM / Vault via data sources, or better, have the application fetch them itself
5. Encrypt state at rest (S3 SSE-KMS) and restrict bucket access — **the state file contains every secret in plaintext**

> **Personal rule, learned the hard way:** anything that enters Terraform enters the state file. Design so secrets never enter.

### IAM
- Least privilege for the Terraform execution role — not `AdministratorAccess` because it's convenient
- Separate roles per environment; the dev role cannot touch prod
- OIDC federation in CI, no static keys
- CloudTrail on, alerting on break-glass role usage

### Policy as code
```bash
tfsec .           # security-focused static analysis
checkov -d .      # broader policy scanning, multi-framework
terrascan scan    # alternative
```
Catches: unencrypted buckets, `0.0.0.0/0` on SSH, public RDS, missing versioning, unencrypted EBS.

**Run these in CI as a gate.** This is on my roadmap gap list and worth closing — it's a differentiator and takes an afternoon.

**Sentinel** (TFC Enterprise) and **OPA** enforce org policy at the plan stage — "all resources must have a CostCenter tag", "no instance larger than m5.xlarge outside prod".

### Repo hygiene — the `.gitignore`

The course's gitignore, with corrections:

```gitignore
**/.terraform/*        # provider binaries, huge, machine-specific
*.tfstate              # CONTAINS SECRETS
*.tfstate.*
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc

# *.tfvars             ← DECIDE DELIBERATELY. Exclude if they hold secrets.
```

**Two problems with the course version:**

1. **It ignores `.terraform.lock.hcl` — this is wrong.** The lock file records exact provider versions and hashes and **should be committed**. It's what guarantees every developer and every CI run resolves identical provider versions. Excluding it defeats its entire purpose. This is a real mistake worth knowing about, because it's a common one.

2. **The tfvars exclusion is commented out.** That's a deliberate decision either way, but it must be a decision. Non-secret environment tfvars are usually fine to commit; anything with a credential is not.

---

## General best practices

| Practice | Why |
|---|---|
| Pin provider versions with `~>` | Patches yes, breaking changes no |
| **Commit `.terraform.lock.hcl`** | Reproducible provider resolution across the team |
| Remote state + locking | Non-negotiable for more than one person |
| Split state by lifecycle/ownership | Plan speed, blast radius, independent locks |
| Modules for anything used twice | Stop copy-pasting |
| `terraform fmt` in CI | Ends formatting arguments permanently |
| Consistent naming + mandatory tags | Cost allocation, ownership, incident response |
| `prevent_destroy` on stateful resources | Accident guardrail |
| Plan output reviewed in the PR | The change is visible before it happens |
| Never run apply from a laptop in prod | Everything through the pipeline |
| Read-only console access in prod | Drift prevention at the source |
| Avoid `-target` routinely | If you need it often, split the config instead |

**Tagging strategy** — use `default_tags` on the provider so you don't repeat yourself:
```hcl
provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Repository  = "github.com/aakash-1004/terraform-fullstack"
    }
  }
}
```
`ManagedBy = terraform` is genuinely useful — it tells anyone in the console not to touch the resource manually, and it makes finding untagged shadow infrastructure trivial.

---

## Interview Q&A

**Q: How would you run Terraform in a CI/CD pipeline?**
> Plan on pull request, apply on merge. On the PR the pipeline runs fmt check, init, validate, a security scan with tfsec or checkov, then `terraform plan -out=tfplan` and posts the plan output as a PR comment so reviewers see exactly what will change. On merge, apply runs against that saved plan file rather than re-planning — that's the important detail, because it guarantees what executes is what was reviewed, with no possibility of drift in between. For authentication I'd use OIDC federation so the pipeline assumes an IAM role and gets short-lived credentials, with the trust policy scoped to the specific repository and branch, rather than storing long-lived access keys as CI secrets.

**Q: What is Terraform Cloud and when would you use it over your own pipeline?**
> It's HashiCorp's managed service — remote state, VCS integration, automated runs, per-workspace variables and credentials, approval gates, run history, and policy enforcement through Sentinel. I'd use it when a team wants the whole workflow managed and doesn't want to build and maintain state infrastructure and pipeline plumbing themselves. I'd use my own CI instead when there are already established pipelines and the team wants everything in one system, or when there's a reason to avoid another vendor. Both are common in practice — the thing to be clear about is that Terraform Cloud workspaces are a completely different concept from CLI workspaces, with their own variables and credentials, which is why they actually give you environment isolation.

**Q: How do you handle secrets in Terraform?**
> The starting point is that the state file contains every attribute value in plaintext, including anything marked sensitive — `sensitive = true` only masks CLI output, it doesn't encrypt state. So the design goal is keeping secrets out of Terraform entirely. Where I can, the application fetches its own secrets at runtime from Secrets Manager or Parameter Store, and Terraform only manages the secret's container and the IAM permission to read it. Where Terraform genuinely needs a value, I pull it with a data source rather than putting it in a tfvars file. And regardless, the state backend gets server-side encryption with KMS and a tight bucket policy, because it's the highest-value target in the whole setup.

**Q: What should and shouldn't be in Git for a Terraform repo?**
> In: all the .tf files, and the `.terraform.lock.hcl` lock file — that one gets excluded by mistake a lot, but it should be committed because it's what pins exact provider versions and hashes so everyone resolves identically. Out: the `.terraform` directory since it's provider binaries, any `.tfstate` files because they contain secrets in plaintext, crash logs, and CLI config files. Tfvars are a judgement call — environment tfvars with non-sensitive values are fine to commit and actually useful for review, but anything holding a credential stays out.

**Q: Where does Terraform stop and Kubernetes tooling start?**
> Terraform owns the infrastructure layer — the VPC, the EKS cluster and node groups, IAM roles including IRSA, databases, load balancers. Helm and ArgoCD own the workload layer — deployments, services, ingresses, application config. Terraform does have Kubernetes and Helm providers, but I'd generally avoid using them for application workloads, because applications deploy many times a day and that churn doesn't belong in an infrastructure state file where a lock blocks everyone else. Keeping the boundary clean means the platform changes slowly through Terraform PRs while applications flow through a GitOps loop. That's exactly how my own repos are split — one Terraform repo for the AWS layer and a separate GitOps repo for the workloads.

**Q: How do you enforce standards across a team's Terraform?**
> Layered. `terraform fmt -check` in CI removes formatting from code review entirely. Static analysis with tfsec or checkov catches security issues like unencrypted buckets or SSH open to the world, run as a gate not a suggestion. Variable validation blocks enforce input rules at the configuration level — approved instance types, required naming patterns. Provider `default_tags` makes mandatory tagging automatic rather than something people remember. And above all that, if the organisation is on Terraform Cloud Enterprise, Sentinel or OPA policies enforce things at plan time across every workspace, like requiring a cost centre tag or blocking oversized instances outside production.

---

## Gotchas & flags

- **`.terraform.lock.hcl` should be committed.** The course gitignore excludes it — that's wrong.
- `sensitive = true` does **not** encrypt state.
- Applying a fresh plan in CI instead of a saved plan file defeats the point of PR review.
- Terraform's `kubernetes`/`helm` providers create a chicken-and-egg problem: the provider config depends on cluster outputs that don't exist until apply. Split into separate configs or use a GitOps tool.
- `-auto-approve` in a pipeline is fine; on a laptop against prod it isn't.
- Static analysis tools have false positives — they're a gate with an exception process, not an oracle.

---

Next: [[18-Interview-Question-Bank]]
