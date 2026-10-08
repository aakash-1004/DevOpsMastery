---
title: "Git Part 4 — Remotes & Collaboration"
type: reference
module: git
tags:
  - git
order: 4
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-03-Branching-and-Merging]]"
next: "[[Git-Part-05-Undo-Recovery-and-History-Rewriting]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 4 — Remotes & Collaboration

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-03-Branching-and-Merging|Git Part 3 — Branching & Merging]] · → [[Git-Part-05-Undo-Recovery-and-History-Rewriting|Git Part 5 — Undo, Recovery & History Rewriting]]
<!-- /nav -->

## 17. Remotes

```bash
git remote -v                             # list remotes with URLs
git remote add origin git@github.com:u/r.git
git remote add upstream https://github.com/original/repo.git
git remote remove <name>
git remote rename origin old-origin
git remote set-url origin <new-url>       # e.g. switch HTTPS → SSH
git remote show origin                    # detailed: tracked branches, push/pull config
git remote prune origin                   # remove stale remote-tracking refs
```

## 18. fetch, pull, push

```bash
git fetch origin                   # download objects + update origin/* refs. Touches NOTHING local.
git fetch --all
git fetch --prune                  # also delete remote-tracking refs for branches deleted upstream
git fetch --tags

git pull                           # = fetch + merge
git pull --rebase                  # = fetch + rebase
git pull --ff-only                 # refuse to create a merge commit — fail instead

git push origin main
git push -u origin feature/x       # push and set upstream tracking
git push --all
git push --tags                    # tags are NOT pushed by default
git push origin --delete feature/x
git push --force-with-lease        # safe force push
git push --force                   # dangerous — avoid
```

**`fetch` vs `pull` — asked constantly.** `fetch` is read-only and always safe: it updates your remote-tracking refs (`origin/main`) so you can inspect what changed. `pull` is `fetch` followed by an automatic merge or rebase into your current branch, which can create conflicts or merge commits you didn't expect.

In scripts and CI, prefer `fetch` + an explicit merge/reset so behaviour is deterministic.

**`--force` vs `--force-with-lease`**

- `--force` overwrites the remote unconditionally. If a colleague pushed in the meantime, their work is gone from the branch tip.
- `--force-with-lease` first checks that the remote branch is still where you last saw it. If someone else pushed, it **refuses**. Use this always.
- `--force-if-includes` (newer) adds a further check that your local history actually contains the remote tip you're overwriting.

Mentioning `--force-with-lease` unprompted reads as senior.

## 19. Tracking branches

```bash
git branch -vv                                # show which upstream each branch tracks
git branch --set-upstream-to=origin/main main
git push -u origin feature/x                  # sets upstream on first push
```

A local branch with an upstream lets you run bare `git pull` / `git push` and see "ahead 2, behind 1" in `git status`.

## 20. SSH vs HTTPS authentication

||HTTPS|SSH|
|---|---|---|
|URL|`https://github.com/u/r.git`|`git@github.com:u/r.git`|
|Auth|Personal Access Token (PAT) — password auth was removed in 2021|SSH key pair|
|Firewalls|Usually allowed (443)|Port 22 may be blocked; use `ssh.github.com:443`|
|CI usage|PAT or `GITHUB_TOKEN`|Deploy key|
|Best for|Ephemeral/CI environments|Developer workstations|

```bash
ssh-keygen -t ed25519 -C "you@example.com"   # why: ed25519 is the modern default; RSA only for legacy
cat ~/.ssh/id_ed25519.pub                     # why: paste into GitHub → Settings → SSH keys
ssh -T git@github.com                         # why: verify auth works
```

**Never commit a PAT or private key.** If you do, rotate it immediately — see Section 25.
