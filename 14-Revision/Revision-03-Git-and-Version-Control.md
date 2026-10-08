---
title: "Revision — Git & Version Control"
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 3
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-02-Linux-and-Bash-Scripting]]"
next: "[[Revision-04-Docker]]"
related:
  - "[[Git-MOC]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Git & Version Control

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-02-Linux-and-Bash-Scripting|Revision — Linux & Bash Scripting]] · → [[Revision-04-Docker|Revision — Docker]]
<!-- /nav -->

## Concept

Git is distributed version control. Every clone is a full repository. This means you can work offline, branch freely, and merge from any source.

**The mental model:** Git tracks snapshots of your project, not diffs. Every commit is a complete snapshot. The diff you see is computed on the fly between two snapshots.

**Key objects:**

- **Blob** — file content
- **Tree** — directory structure
- **Commit** — snapshot + metadata + parent pointer
- **Tag** — named pointer to a commit

---

## Core Commands

```bash
# Daily workflow
git status                            # What's changed, what's staged
git add -p                            # Interactive staging — review hunks before adding
git commit -m "feat: add health check"
git push origin feature/my-feature

# Branching
git checkout -b feature/new-thing     # Create and switch to new branch
git branch -d feature/done            # Delete local branch
git push origin --delete feature/done # Delete remote branch

# Merging & rebasing
git merge feature/new-thing           # Merge with merge commit (preserves history)
git rebase main                       # Replay commits on top of main (linear history)
git rebase -i HEAD~3                  # Interactive rebase: squash/edit last 3 commits

# Conflict resolution
git status                            # Shows conflicted files
# Edit conflicted files, remove <<<<< ===== >>>>> markers
git add <resolved-file>
git rebase --continue                 # Or: git merge --continue

# Undoing things
git reset --soft HEAD~1               # Undo last commit, keep changes staged
git reset --mixed HEAD~1              # Undo last commit, keep changes unstaged
git reset --hard HEAD~1               # Undo last commit, DISCARD changes (dangerous)
git revert <commit-hash>              # Create new commit that undoes a commit (safe for shared branches)

# Inspection
git log --oneline --graph --all       # Visual branch graph
git diff HEAD~1 HEAD                  # Diff last commit vs current
git stash                             # Temporarily shelve changes
git stash pop                         # Restore stashed changes

# SSH key setup
ssh-keygen -t ed25519 -C "aakash@zehntech.com"
cat ~/.ssh/id_ed25519.pub             # Copy this to GitHub → Settings → SSH Keys
ssh -T git@github.com                 # Test SSH connection
```

---

## Senior Interview Q&A

**Q: Merge vs rebase — when do you use each?**

> "The decision comes down to history vs simplicity.
>
> Merge preserves the full history — you can see exactly when a feature branch diverged and when it was integrated. It creates a merge commit. This is valuable for audit trails and understanding how features were developed in parallel. Use merge for integrating long-lived feature branches into main, and especially for things like release branches where you want a clear record.
>
> Rebase replays your commits on top of the target branch, creating a linear history. It looks like the work was done sequentially even if it wasn't. This makes `git log` much cleaner and makes `git bisect` more powerful because history is linear.
>
> My rule: rebase your personal feature branches before opening a PR. Never rebase shared branches that others have pulled — you're rewriting history and everyone else's branch will diverge. For shared branches, always merge.
>
> In practice at most teams: developers rebase locally to clean up their commits, then the PR merge uses a squash merge or a regular merge commit into main."

---

**Q: What's `git reset --soft` vs `--hard` and when would you use them in production?**

> "All three reset modes move HEAD to a different commit. The difference is what they do with your working tree and staging area.
>
> `--soft` moves HEAD only. Your changes are still staged. Use this when you made a commit but want to restructure it — maybe split it into two commits, or fix the commit message, or add a file you forgot.
>
> `--mixed` (the default) moves HEAD and unstages changes. Your working tree is unchanged. Use this when you want to recommit differently.
>
> `--hard` moves HEAD, unstages, AND discards working tree changes. This is destructive. Use it when you want to completely abandon work and return to a known state.
>
> In production context: I'd never `git reset --hard` on a shared branch. If something went to main that shouldn't have, I use `git revert` — it creates a new commit that undoes the bad one, preserving history. `reset --hard` rewrites history, which breaks everyone who has pulled. `revert` is the production-safe tool."

---

**Q: How do you structure a Git branching strategy for a team?**

> "It depends on deployment cadence. Two main models I've used:
>
> **GitFlow** for teams with scheduled releases: main (production), develop (integration), feature/* (features), release/* (stabilization), hotfix/* (emergency fixes). Good for mobile apps or enterprise software where releases are versioned. Downside: complex, long-lived branches, merge conflicts.
>
> **Trunk-based development** for teams doing continuous deployment: everyone branches off main, keeps branches short-lived (1-2 days max), merges back frequently. Feature flags hide incomplete features in production. This is what Google, Netflix, and Etsy use. It requires good CI and feature flags to work.
>
> My recommendation for most teams: trunk-based with short feature branches, squash merges into main, and automated pipelines triggered on main. It scales better and forces you to integrate continuously rather than doing a big bang merge at the end."

<!-- related -->
---
**Related:** [[Git-MOC|Git & GitHub]]
<!-- /related -->
