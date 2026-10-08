---
title: "Git Part 6 — Tags, Submodules, Hooks, Worktrees, LFS"
type: reference
module: git
tags:
  - git
order: 6
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-05-Undo-Recovery-and-History-Rewriting]]"
next: "[[Git-Part-07-GitHub]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 6 — Tags, Submodules, Hooks, Worktrees, LFS

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-05-Undo-Recovery-and-History-Rewriting|Git Part 5 — Undo, Recovery & History Rewriting]] · → [[Git-Part-07-GitHub|Git Part 7 — GitHub]]
<!-- /nav -->

## 28. Tags and releases

```bash
git tag                                  # list
git tag -l "v1.*"                        # filter
git tag v1.0.0                           # LIGHTWEIGHT — just a ref, no object
git tag -a v1.0.0 -m "Release 1.0.0"     # ANNOTATED — a real object with tagger/date/message
git tag -s v1.0.0 -m "Signed release"    # GPG-signed
git tag -a v1.0.0 <sha>                  # tag a past commit
git show v1.0.0                          # tag object + the commit it points to
git push origin v1.0.0                   # tags are NOT pushed by default
git push origin --tags                   # push all tags
git tag -d v1.0.0                        # delete locally
git push origin --delete v1.0.0          # delete remotely
git describe --tags --always --dirty     # v1.4.2-3-gabc1234-dirty
```

**Always use annotated tags for releases.** They record who/when/why and can be signed. Lightweight tags create no object at all.

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH        e.g. 2.4.1
  │     │     └── backwards-compatible bug fixes
  │     └──────── backwards-compatible new features
  └────────────── breaking changes
Pre-release: 2.4.1-rc.1, 2.4.1-beta.2
Build metadata: 2.4.1+20260928.abc1234
```

**DevOps use:** CI tags container images from Git. Common patterns:

```bash
docker build -t myapp:$(git rev-parse --short HEAD) .   # immutable SHA tag
docker build -t myapp:$(git describe --tags --always) . # human-readable release tag
```

A tag push (`refs/tags/v*`) typically triggers the release pipeline, while a branch push triggers build/test.

## 29. Submodules

A submodule is a pointer to a **specific commit** in another repository, recorded as a `160000` gitlink entry in the tree.

```bash
git submodule add https://github.com/u/lib.git libs/lib
git clone --recurse-submodules <url>          # clone including submodules
git submodule update --init --recursive       # after a normal clone
git submodule update --remote                 # pull the latest from each submodule's branch
git submodule foreach 'git pull origin main'
git submodule status
```

`.gitmodules` (committed) holds the URL and path; the tree holds the pinned SHA.

**Pain points:** submodules check out in detached HEAD; forgetting `--recurse-submodules` gives an empty directory; CI must be configured to fetch them; updating requires a commit in the parent repo.

### Submodule vs subtree

||Submodule|Subtree|
|---|---|---|
|Storage|Pointer to an external repo|Files vendored into your repo|
|Clone UX|Needs `--recurse-submodules`|Just works|
|History|Separate|Merged into yours|
|Upstream contribution|Natural|Needs `git subtree push`|
|Repo size|Small|Larger|

```bash
git subtree add --prefix=libs/lib https://github.com/u/lib.git main --squash
git subtree pull --prefix=libs/lib https://github.com/u/lib.git main --squash
```

**Recommendation:** avoid both for shared code. Publish a **versioned artifact** instead — a Helm chart in a chart repo, a Go module, a Terraform module in a registry, a container image. Submodules are acceptable for pinning a config/manifest repo where you genuinely want an exact commit.

## 30. Hooks

Scripts that run at defined points in the Git lifecycle. They live in `.git/hooks/` and are **not versioned or cloned**.

### Client-side hooks

|Hook|Fires|Typical use|
|---|---|---|
|`pre-commit`|Before the commit is created|Lint, format, secret scan, run fast tests|
|`prepare-commit-msg`|Before the message editor opens|Inject a ticket ID from the branch name|
|`commit-msg`|After the message is written|Enforce Conventional Commits|
|`post-commit`|After commit|Notifications|
|`pre-push`|Before pushing|Run the test suite, block pushes to main|
|`post-checkout` / `post-merge`|After switching/merging|Reinstall dependencies|

### Server-side hooks

|Hook|Use|
|---|---|
|`pre-receive`|Reject the whole push (policy enforcement)|
|`update`|Per-branch acceptance check|
|`post-receive`|Trigger deployment, notify CI, post to Slack|

### Making hooks shareable

Local hooks aren't versioned and are trivially bypassed with `git commit --no-verify`. Two fixes:

```bash
git config core.hooksPath .githooks     # why: point Git at a versioned directory in the repo
```

or use the **`pre-commit` framework** (`.pre-commit-config.yaml`, committed):

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks: [{id: gitleaks}]
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.86.0
    hooks: [{id: terraform_fmt}, {id: terraform_validate}, {id: terraform_tflint}]
```

**Key principle:** client-side hooks are a _convenience_ for developers. **Real enforcement belongs in CI and branch protection rules**, because local hooks are optional and bypassable.

## 31. Worktrees

Check out multiple branches simultaneously into separate directories, sharing one `.git`.

```bash
git worktree add ../hotfix hotfix/urgent    # why: work on a hotfix without stashing your feature work
git worktree add -b newbranch ../dir
git worktree list
git worktree remove ../hotfix
git worktree prune
```

Far cleaner than cloning the repo twice — history is shared, disk usage stays low. Useful in CI when you need to build two refs side by side.

## 32. Git LFS (Large File Storage)

Replaces large binaries with small text pointers in Git; the actual bytes live on a separate LFS server.

```bash
git lfs install
git lfs track "*.psd" "*.zip" "*.mp4"
git add .gitattributes
git lfs ls-files
```

Use for assets, ML model weights, large test fixtures. GitHub blocks files over 100 MB in normal Git; warns at 50 MB.

**Better DevOps practice:** don't put build artifacts in Git at all. Use S3, Artifactory, Nexus, ECR or GitHub Packages.
