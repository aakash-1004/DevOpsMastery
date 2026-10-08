---
title: Git Part 12 — Cheat Sheet
type: reference
module: git
tags:
  - git
order: 12
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-11-Interview-Question-Bank]]"
next: "[[Git-Part-13-What-Interviewers-Actually-Test]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 12 — Cheat Sheet

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-11-Interview-Question-Bank|Git Part 11 — Interview Question Bank]] · → [[Git-Part-13-What-Interviewers-Actually-Test|Git Part 13 — What Interviewers Actually Test]]
<!-- /nav -->

```bash
# ── SETUP ──────────────────────────────────────────────
git config --global user.name "Name"
git config --global user.email "a@b.com"
git config --list --show-origin

# ── START ──────────────────────────────────────────────
git init | git init --bare
git clone <url> [dir]
git clone --depth 1 <url>              # shallow (CI)
git clone --filter=blob:none <url>     # partial (CI, keeps history)

# ── DAILY ──────────────────────────────────────────────
git status -s
git add -p | git add -A
git commit -m "msg" | --amend --no-edit
git diff | --staged | --name-only origin/main...HEAD
git log --oneline --graph --decorate --all
git show <sha> | git blame <file>

# ── BRANCH ─────────────────────────────────────────────
git switch -c feat/x | git switch -
git branch -vv | -a | --merged | -d | -D
git push -u origin feat/x
git push origin --delete feat/x

# ── INTEGRATE ──────────────────────────────────────────
git merge feat | --no-ff | --squash | --abort
git rebase main | -i HEAD~4 | --continue | --abort
git cherry-pick -x <sha>
git merge-base main feat

# ── SYNC ───────────────────────────────────────────────
git fetch --all --prune
git pull --rebase | --ff-only
git push --force-with-lease
git remote -v | add | set-url | show origin

# ── UNDO ───────────────────────────────────────────────
git restore <file>              # discard working changes
git restore --staged <file>     # unstage
git reset --soft|--mixed|--hard HEAD~1
git revert <sha> | -m 1 <merge-sha>
git reflog && git reset --hard HEAD@{n}
git stash push -m "wip" -u | pop | list

# ── FIND ───────────────────────────────────────────────
git log -S "string"             # pickaxe
git log --grep="msg"
git bisect start / bad / good / run ./t.sh / reset
git fsck --lost-found

# ── TAGS ───────────────────────────────────────────────
git tag -a v1.0.0 -m "msg"
git push origin v1.0.0 | --tags
git describe --tags --always --dirty

# ── INSPECT INTERNALS ──────────────────────────────────
git cat-file -p|-t|-s <sha>
git rev-parse HEAD | --short HEAD
git ls-tree -r HEAD
git count-objects -vH

# ── CLEANUP ────────────────────────────────────────────
git clean -nd    # dry run
git clean -fdx   # nuke untracked + ignored (CI)
git gc --prune=now
```

## Command → intent quick map

|I want to...|Command|
|---|---|
|Undo a commit, keep the work|`git reset --soft HEAD~1`|
|Undo a pushed commit safely|`git revert <sha>`|
|Discard local file changes|`git restore <file>`|
|Unstage a file|`git restore --staged <file>`|
|Recover after `reset --hard`|`git reflog` then `git reset --hard HEAD@{n}`|
|Move a commit to another branch|`git cherry-pick <sha>`|
|Clean up my commits before a PR|`git rebase -i HEAD~n`|
|Find the commit that broke it|`git bisect run ./test.sh`|
|Find which commit added a string|`git log -S "string"`|
|See what's on my branch vs main|`git log main..feature`|
|Update my branch from main|`git rebase main` (unshared) or `git merge main`|
|Force push without stomping someone|`git push --force-with-lease`|
|Untrack a file but keep it|`git rm --cached <file>`|
|Why is this file ignored?|`git check-ignore -v <file>`|
|Reset CI workspace|`git clean -fdx && git reset --hard`|
