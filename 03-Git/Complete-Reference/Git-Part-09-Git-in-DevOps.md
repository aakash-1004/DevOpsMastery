---
title: Git Part 9 — Git in DevOps
type: reference
module: git
tags:
  - git
order: 9
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-08-Branching-Strategies]]"
next: "[[Git-Part-10-Advanced-Internals-and-Performance]]"
related:
  - "[[7.1-GitHub-Actions]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 9 — Git in DevOps

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-08-Branching-Strategies|Git Part 8 — Branching Strategies]] · → [[Git-Part-10-Advanced-Internals-and-Performance|Git Part 10 — Advanced Internals & Performance]]
<!-- /nav -->

## 40. Git in CI/CD pipelines

```bash
# Fast checkouts
git clone --depth 1 <url>                     # why: shallow clone — only the tip commit
git clone --depth 1 --branch v1.2.0 <url>     # why: shallow clone of a specific tag
git clone --filter=blob:none <url>            # why: blobless clone — full history, files fetched lazily
git clone --filter=tree:0 <url>               # why: treeless — even lighter, good for CI
git clone --single-branch --branch main <url> # why: skip all other branches

# Monorepo
git sparse-checkout init --cone
git sparse-checkout set services/api charts/  # why: materialise only the paths you need
git diff --name-only origin/main...HEAD       # why: changed-file detection to decide which pipelines run

# Metadata for tagging artifacts
git rev-parse --short HEAD                    # why: immutable image tag
git describe --tags --always --dirty          # why: human-readable version
git log -1 --pretty=%s                        # why: commit subject for notifications
git rev-list --count HEAD                     # why: monotonic build number

# Workspace hygiene between builds
git clean -fdx && git reset --hard            # why: guarantee a pristine workspace
```

**Shallow clone caveat:** `--depth 1` breaks `git describe`, `git log --follow`, SonarQube blame, and anything needing history. In GitHub Actions use `fetch-depth: 0` when you need full history.

## 41. GitOps

**Definition:** an operating model where **Git is the single source of truth for declarative infrastructure and application state**, and an in-cluster agent continuously reconciles the live system to match the repo.

Four principles:

1. The system is described **declaratively**
2. Desired state is **versioned in Git**
3. Approved changes are **applied automatically**
4. An agent **continuously reconciles** and corrects drift

Tools: **ArgoCD**, **Flux**.

### Why Git is a good fit

A commit SHA is a **cryptographically fixed, complete snapshot of desired state**. ArgoCD can report "cluster is synced to `abc123`" and that means exactly one tree of manifests, unambiguously. This is why commit immutability matters in practice.

### Repo structure — app repo vs config repo

**Separate the application source repo from the manifests/config repo.** Reasons:

- A CI job committing a new image tag into the _same_ repo would retrigger CI — an infinite loop
- Different RBAC: developers own app code, platform team owns cluster config
- Deployment history stays clean and auditable, independent of code churn
- One config repo can deploy artifacts from many app repos

Typical config repo layout:

```
config-repo/
├── apps/
│   ├── myapp/
│   │   ├── base/                 # Kustomize base or Helm chart
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
└── clusters/
    ├── dev/
    └── prod/
```

### Flow

```
dev pushes code → CI builds & tests → pushes image tagged with git SHA →
CI commits new image tag to config repo → ArgoCD detects the commit →
syncs cluster → reports Synced/Healthy
```

### Rollback in GitOps

**Rollback = `git revert`.** Revert the config commit; the agent reconciles the cluster back. Your deployment history _is_ your Git history, and it's fully auditable — who changed what, when, and who approved it via the PR.

### Secrets in GitOps

Never put plain secrets in Git. Options:

- **Sealed Secrets** (Bitnami) — encrypt with a cluster-specific key; the SealedSecret is safe to commit
- **External Secrets Operator** — pull from AWS Secrets Manager / Vault at runtime
- **SOPS + age/KMS** — encrypt values in-place in YAML
- Out-of-band `kubectl create secret` — simplest; acceptable if the trade-off is deliberate and documented

### Drift detection

The agent continuously diffs live cluster state against Git. Manual `kubectl edit` changes get flagged as **OutOfSync** and, with self-heal enabled, reverted automatically. This is the direct analogue of Terraform drift.

## 42. Git and Infrastructure as Code

- Terraform, Ansible, Helm charts, Kubernetes manifests — all belong in Git, reviewed via PR.
- `.gitignore` for Terraform:

    ```gitignore
    .terraform/*.tfstate*.tfstate.**.tfvars          # often contains secretscrash.log
    ```

    **Commit `.terraform.lock.hcl`** — it pins provider versions. **Never commit `*.tfstate`** — it contains secrets in plain text, and Git provides no locking. Use an S3 backend with DynamoDB (or S3 native) locking.
- PR-time checks for IaC: `terraform fmt -check`, `terraform validate`, `tflint`, `checkov`, `tfsec`, and posting `terraform plan` output as a PR comment (Atlantis, or a GitHub Actions job).

## 43. Supply-chain security with Git

Increasingly asked at senior level.

- **Signed commits and tags** — `git commit -S`, `git tag -s`. Proves authorship. Enforce with "Require signed commits" branch protection.

    ```bash
    git config --global user.signingkey <key-id>git config --global commit.gpgsign truegit config --global gpg.format ssh          # sign with an SSH key instead of GPG
    ```

- **Pin third-party GitHub Actions to a full commit SHA**, not a mutable tag.
- **Secret scanning + push protection** enabled org-wide.
- **Dependabot / Renovate** for dependency updates.
- **SBOM generation** in the pipeline (Syft, `cyclonedx`), image signing with **cosign**, provenance attestation with **SLSA**.
- **Protected branches + CODEOWNERS** so no single person can push to production paths unreviewed.

<!-- related -->
---
**Related:** [[7.1-GitHub-Actions|GitHub Actions — CI/CD]]
<!-- /related -->
