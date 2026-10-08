---
title: Git Part 7 — GitHub
type: reference
module: git
tags:
  - git
order: 7
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-06-Tags-Submodules-Hooks-Worktrees-LFS]]"
next: "[[Git-Part-08-Branching-Strategies]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 7 — GitHub

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-06-Tags-Submodules-Hooks-Worktrees-LFS|Git Part 6 — Tags, Submodules, Hooks, Worktrees, LFS]] · → [[Git-Part-08-Branching-Strategies|Git Part 8 — Branching Strategies]]
<!-- /nav -->

## 33. Repositories, forks and pull requests

**Repository visibility:** public, private, internal (Enterprise only).

**Fork** — a server-side copy of a repository under your own account. A GitHub concept, not a Git one. Used when you don't have write access to the original.

```bash
# Fork workflow
git clone git@github.com:you/repo.git
git remote add upstream https://github.com/original/repo.git   # why: track the source repo
git fetch upstream
git merge upstream/main          # why: keep your fork current
git push origin main
```

**Pull Request (PR)** — a request to merge one branch into another, with a review and discussion layer on top. GitLab calls it a Merge Request (MR).

Typical PR lifecycle:

1. Branch from `main`
2. Commit and push
3. Open a PR, fill the template, link the issue (`Closes #123`)
4. CI runs required status checks
5. Reviewers comment / request changes / approve
6. Merge via one of the three strategies below
7. Branch auto-deleted

### GitHub's three merge buttons

|Button|Result|When to use|
|---|---|---|
|**Create a merge commit** (`--no-ff`)|Keeps every commit + adds a merge commit|Release branches, when full history matters|
|**Squash and merge**|All PR commits collapse into one on main|**Default choice** for feature PRs — clean history, easy revert/bisect|
|**Rebase and merge**|Commits replayed onto main, no merge commit|Linear history where each commit is meaningful|

### Draft PRs, auto-merge, merge queue

- **Draft PR** — signals work in progress; cannot be merged.
- **Auto-merge** — merges automatically once all checks pass and approvals land.
- **Merge queue** — serialises merges and re-tests each PR against the latest main, preventing "semantic conflicts" where two PRs pass individually but break together. Important at scale.

### Linking issues

`Closes #12`, `Fixes #12`, `Resolves #12` in a PR body or commit message auto-closes the issue on merge.

## 34. Branch protection and repository rules

Configured under Settings → Branches (or the newer **Rulesets**).

Common protections on `main`:

- Require a pull request before merging (N approvals)
- Dismiss stale approvals when new commits are pushed
- Require review from **Code Owners**
- Require status checks to pass (name the CI jobs)
- Require branches to be up to date before merging
- Require conversation resolution
- Require signed commits
- Require linear history (blocks merge commits)
- Block force pushes
- Block deletions
- Include administrators (otherwise admins bypass everything)

This is the _real_ enforcement layer — not client-side hooks.

### CODEOWNERS

`.github/CODEOWNERS` — path-based automatic reviewer assignment.

```
*                       @org/platform-team
/terraform/             @org/cloud-infra
/.github/workflows/     @org/devops
/charts/                @org/devops @org/sre
*.tf                    @aakash-1004
/docs/                  @org/tech-writers
```

Combined with "Require review from Code Owners", this makes infrastructure changes impossible to merge without the right team's sign-off. A strong thing to mention in interviews.

## 35. Issues, projects and templates

- **Issues** — bug/feature tracking with labels, milestones, assignees.
- **Projects (v2)** — boards/tables/roadmaps built on issues and PRs.
- **Discussions** — forum-style Q&A.
- **Templates:**
    - `.github/ISSUE_TEMPLATE/bug_report.yml`
    - `.github/PULL_REQUEST_TEMPLATE.md`
    - `.github/CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`

## 36. GitHub Actions (CI/CD)

Event-driven CI/CD built into GitHub. Workflows live in `.github/workflows/*.yml`.

### Hierarchy

```
Workflow  →  Job(s)  →  Step(s)  →  Action or shell command
```

- **Workflow** — one YAML file, triggered by events
- **Job** — runs on one runner; jobs are parallel by default, sequenced with `needs`
- **Step** — a single task; `uses:` an action or `run:` a shell command
- **Runner** — the machine (GitHub-hosted `ubuntu-latest` or self-hosted)

### Anatomy

```yaml
name: build-and-deploy

on:
  push:
    branches: [main]
    paths: ['src/**', 'Dockerfile']     # why: path filter avoids pointless runs in a monorepo
  pull_request:
    branches: [main]
  tags: ['v*']
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:                     # why: manual trigger button in the UI
    inputs:
      environment:
        type: choice
        options: [dev, staging, prod]

env:
  AWS_REGION: ap-south-1
  IMAGE: myapp

concurrency:                             # why: cancel superseded runs on the same branch
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:                             # why: least privilege for GITHUB_TOKEN
  contents: read
  id-token: write                        # why: required for OIDC to AWS
  packages: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0                 # why: full history — needed for git describe / SonarQube
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - run: go test ./...

  build:
    needs: test                          # why: run only after 'test' succeeds
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: ${{ env.AWS_REGION }}     # why: OIDC — no long-lived AWS keys in secrets
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tag }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production              # why: enables required reviewers / approval gate
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        region: [ap-south-1, us-east-1]  # why: fan out the same job across values
    steps:
      - run: echo "deploying ${{ needs.build.outputs.image_tag }} to ${{ matrix.region }}"
```

### Key concepts to know

|Concept|Detail|
|---|---|
|**`GITHUB_TOKEN`**|Auto-generated per run, scoped to the repo, expires when the job ends. Set `permissions:` to least privilege.|
|**Secrets**|Repo / environment / organization level. Masked in logs. Not available to PRs from forks by default.|
|**Variables**|Non-sensitive config (`vars.MY_VAR`).|
|**Environments**|Named deploy targets with protection rules — required reviewers, wait timers, branch restrictions. This is how you build an approval gate.|
|**OIDC**|`id-token: write` + `configure-aws-credentials` lets Actions assume an IAM role with **no long-lived AWS keys stored as secrets**. This is the modern best practice — mention it.|
|**Caching**|`actions/cache` for dependencies; `cache-from`/`cache-to` for Docker layers.|
|**Artifacts**|`actions/upload-artifact` / `download-artifact` to pass files between jobs.|
|**Reusable workflows**|`uses: org/repo/.github/workflows/deploy.yml@v1` with `workflow_call`.|
|**Composite actions**|Bundle several steps into one reusable action.|
|**Self-hosted runners**|For private network access (e.g. a VPC-internal EKS cluster) or special hardware. Never use them on public repos — untrusted PR code would run on your infrastructure.|
|**`pull_request` vs `pull_request_target`**|`pull_request_target` runs with repo secrets and the base branch's workflow — **dangerous** if you check out PR code. A well-known security footgun.|
|**Contexts**|`github.*`, `env.*`, `secrets.*`, `vars.*`, `job.*`, `steps.*`, `needs.*`, `matrix.*`, `runner.*`|
|**Action pinning**|Pin third-party actions to a **full commit SHA**, not a tag — tags are mutable, which is a supply-chain risk.|

```bash
# Useful in CI — changed-file detection for monorepos
git diff --name-only origin/main...HEAD
```

## 37. Other GitHub features

|Feature|What it does|
|---|---|
|**GitHub Packages**|Registry for npm, Maven, NuGet, RubyGems, and container images (`ghcr.io`).|
|**Releases**|Attach binaries/notes to a tag. Auto-generated release notes from PR titles.|
|**Dependabot**|Automated dependency update PRs + security alerts. Configured via `.github/dependabot.yml`.|
|**Secret scanning**|Detects committed credentials. **Push protection** blocks the push outright.|
|**Code scanning / CodeQL**|Static analysis for vulnerabilities, results in the Security tab.|
|**Dependency graph / SBOM**|Inventory of dependencies; exportable as SPDX.|
|**GitHub Pages**|Static site hosting from a branch or `/docs`.|
|**Gists**|Standalone snippet repos.|
|**GitHub Apps vs OAuth Apps**|Apps act on their own behalf with fine-grained, per-repo permissions — preferred over PATs for automation.|
|**Deploy keys**|Single-repo SSH keys, read-only by default. Better than a user PAT for CI clone access.|
|**Fine-grained PATs**|Scoped per-repo, per-permission, with expiry. Prefer over classic PATs.|
|**Rulesets**|Newer, layerable replacement for branch protection; can apply org-wide.|
|**Codespaces**|Cloud dev environments defined by `.devcontainer/`.|
|**`git` CLI for GitHub — `gh`**|See below.|

### GitHub CLI

```bash
gh auth login
gh repo clone owner/repo
gh repo create myrepo --private --source=. --push
gh pr create --base main --head feature/x --title "..." --body "..."
gh pr list / gh pr view 42 / gh pr checkout 42
gh pr merge 42 --squash --delete-branch
gh pr review 42 --approve
gh issue create --title "..." --label bug
gh run list / gh run view <id> --log / gh run rerun <id>
gh workflow run deploy.yml -f environment=prod
gh release create v1.0.0 ./dist/* --notes "Release notes"
gh secret set AWS_ROLE_ARN --body "arn:aws:iam::..."
```
