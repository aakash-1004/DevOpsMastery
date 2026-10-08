---
title: "Git Part 10 — Advanced Internals & Performance"
type: reference
module: git
tags:
  - git
order: 10
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-09-Git-in-DevOps]]"
next: "[[Git-Part-11-Interview-Question-Bank]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 10 — Advanced Internals & Performance

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-09-Git-in-DevOps|Git Part 9 — Git in DevOps]] · → [[Git-Part-11-Interview-Question-Bank|Git Part 11 — Interview Question Bank]]
<!-- /nav -->

## 44. Packfiles and garbage collection

Loose objects (one file per object) are inefficient at scale. Git periodically packs them into **packfiles** (`.git/objects/pack/*.pack`) with delta compression between similar objects — this is the only place Git stores deltas, and it's a _storage_ optimisation, not the data model.

```bash
git gc                        # pack loose objects, prune unreachable ones, expire reflog
git gc --aggressive           # slower, better compression
git gc --prune=now            # prune immediately (this is what destroys orphaned commits)
git count-objects -vH         # loose vs packed stats
git verify-pack -v <.idx>     # inspect a packfile
git repack -ad                # repack everything into a single pack
git prune                     # remove unreachable objects
```

`gc` runs automatically after certain commands when loose objects exceed a threshold.

## 45. Large repository performance

```bash
git config core.fsmonitor true        # filesystem monitor — much faster status on big repos
git config core.untrackedCache true
git config feature.manyFiles true
git maintenance start                 # schedule background prefetch/repack
git commit-graph write --reachable    # speeds up log/merge-base on huge histories
```

Also: partial clone (`--filter=blob:none`), sparse checkout, and shallow clones in CI.

## 46. Useful plumbing commands

Porcelain = user-facing. Plumbing = low-level, script-friendly.

```bash
git hash-object -w <file>      # write a blob, print its SHA
git cat-file -p / -t / -s      # print / type / size of any object
git ls-tree -r HEAD            # recursive listing of a tree
git ls-files -s                # index contents with modes and SHAs
git rev-parse <ref>            # resolve any ref to a SHA
git rev-list --all             # every commit SHA
git symbolic-ref HEAD          # what HEAD points to
git update-ref refs/heads/x <sha>   # move a ref directly
git merge-base main feature    # common ancestor of two branches
git write-tree / git commit-tree    # build objects manually
```

## 47. Miscellaneous advanced features

```bash
# rerere — remember and replay conflict resolutions
git config --global rerere.enabled true

# Bundle — move a repo across an air gap as a single file
git bundle create repo.bundle --all
git clone repo.bundle myrepo

# Notes — attach metadata to a commit without changing its SHA
git notes add -m "deployed to prod 2026-09-28" <sha>
git log --show-notes

# Replace — graft history without rewriting
git replace <old-sha> <new-sha>

# Apply a patch by email / file
git format-patch -1 <sha>
git am < patch.eml
git apply changes.patch

# Split a subdirectory into its own repo
git subtree split --prefix=services/api -b api-only

# Interactive staging
git add -i
```
