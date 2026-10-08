---
title: "Git Part 3 — Branching & Merging"
type: reference
module: git
tags:
  - git
order: 3
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-02-Internals]]"
next: "[[Git-Part-04-Remotes-and-Collaboration]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 3 — Branching & Merging

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-02-Internals|Git Part 2 — Internals]] · → [[Git-Part-04-Remotes-and-Collaboration|Git Part 4 — Remotes & Collaboration]]
<!-- /nav -->

## 12. Branches

A branch is a **movable pointer to a commit**. Creating one writes 41 bytes. That's why Git branching is cheap and why feature-branch and trunk-based workflows are practical.

```bash
# Create / list / switch
git branch                          # list local branches (* = current)
git branch -a                       # include remote-tracking branches
git branch -r                       # remote-tracking only
git branch -v                       # show last commit on each
git branch --merged                 # branches already merged into HEAD (safe to delete)
git branch --no-merged              # branches with unmerged work

git branch feature/login            # create, don't switch
git switch feature/login            # switch (modern, Git 2.23+)
git switch -c feature/login         # create + switch
git checkout -b feature/login       # older equivalent
git switch -                        # switch back to previous branch

# Delete
git branch -d feature/login         # safe delete — refuses if unmerged
git branch -D feature/login         # force delete
git push origin --delete feature/login   # delete on the remote

# Rename
git branch -m old-name new-name
git branch -m new-name              # rename current branch
```

**`checkout` vs `switch` vs `restore`** `git checkout` was overloaded — it switched branches _and_ discarded file changes, which caused accidental data loss. Git 2.23 split it:

- `git switch` — change branches only
- `git restore` — restore file contents only
- `git checkout` — still works, does both

Use `switch`/`restore` in new material; recognise `checkout` because it's everywhere in older docs.

### Detached HEAD

```bash
git checkout <sha>          # HEAD now points at a commit, not a branch
```

Commits made here belong to no branch. Switching away orphans them — recoverable via `git reflog`, until GC. To keep the work:

```bash
git switch -c rescue-branch      # why: attach a branch name to the current detached commit
```

CI systems frequently check out a detached SHA — that's normal and intentional.

## 13. Merging

```bash
git switch main
git merge feature                   # merge feature into main
git merge --no-ff feature           # always create a merge commit
git merge --squash feature          # collapse all feature commits into one staged change
git merge --abort                   # bail out of an in-progress merge
git merge --continue                # after resolving conflicts
```

### Three kinds of merge

**1. Fast-forward** — target branch hasn't moved since the feature branched. Git just moves the pointer forward. No merge commit, linear history.

```
before:  A---B---C (main)
                  \
                   D---E (feature)
after:   A---B---C---D---E (main, feature)
```

**2. Three-way merge** — both branches moved. Git finds the **merge base** (nearest common ancestor), compares both tips against it, and creates a **merge commit with two parents**.

```
before:  A---B---C---F (main)
                  \
                   D---E (feature)
after:   A---B---C---F---M (main)
                  \     /
                   D---E
```

**3. Squash merge** — all feature commits are collapsed into a single new commit on main. The feature branch's individual commits are not part of main's history. This is the default in most modern PR workflows.

### Which to use — concrete recommendation

|Situation|Use|
|---|---|
|Short-lived PR branch → main|**Squash merge.** One commit per unit of work. Makes `git revert` and `git bisect` trivial.|
|Long-lived release branch → main|**`--no-ff` merge.** Preserves that a release line existed.|
|Hotfix branch → main and release|**Cherry-pick** the fix commit.|
|Updating your own feature branch from main|**Rebase** (local only, before pushing) or merge if already shared.|

### Merge strategies

- **ort** — default since Git 2.34 (replaced `recursive`). Handles the general two-branch case, including renames and criss-cross merges.
- **octopus** — default when merging more than two branches.
- **ours** — keep our side entirely, record the merge. `git merge -s ours other` — used to mark a branch as merged without taking its content.
- **subtree** — for merging a project into a subdirectory.
- Options: `-X ours` / `-X theirs` — resolve only _conflicting hunks_ in favour of one side (different from the `ours` strategy, which discards the other side entirely).

## 14. Conflicts

A conflict occurs when both branches changed **the same region of the same file**, or one deleted a file the other modified. Git never guesses.

```
<<<<<<< HEAD
image: myapp:v1.2.0
=======
image: myapp:v1.3.0
>>>>>>> feature/bump-image
```

Resolution workflow:

```bash
git status                     # why: lists files as "both modified"
git diff                       # why: shows the conflicted hunks
# edit the file, remove the <<<<<<< ======= >>>>>>> markers, keep the correct content
git add <file>                 # why: staging a conflicted file marks it RESOLVED
git commit                     # why: completes the merge (default message is pre-filled)

# escape hatches
git merge --abort              # why: full rollback to pre-merge state
git rebase --abort
git rebase --skip              # why: drop the current commit during a rebase
```

Helpful tools:

```bash
git checkout --ours  <file>    # why: take our version wholesale
git checkout --theirs <file>   # why: take their version wholesale
git mergetool                  # why: launch a configured 3-way merge GUI
git config merge.conflictStyle zdiff3   # why: also shows the common ancestor — far easier to reason about
git config --global rerere.enabled true # why: REuse REcorded REsolution — replays past conflict fixes automatically
```

`rerere` is a strong senior signal — it's invaluable on long-running branches where you re-resolve the same conflict repeatedly.

**Meaning of "ours" and "theirs" flips during a rebase.** In a merge, ours = current branch. In a rebase, ours = the branch you're replaying _onto_ (upstream), theirs = your commits being replayed. This trips up a lot of engineers.

## 15. Rebase

Rebase takes your commits, rewinds them, and **replays them one by one** on top of a new base. Each replayed commit is a **new object with a new SHA**.

```bash
git switch feature
git rebase main                  # replay feature's commits on top of current main
git rebase --onto main dev topic # replay topic onto main, dropping dev's commits
git rebase -i HEAD~4             # interactive: edit the last 4 commits
git rebase --continue / --abort / --skip
```

```
before:  A---B---C---F (main)
                  \
                   D---E (feature)
after:   A---B---C---F (main)
                      \
                       D'---E' (feature)   ← D' and E' are NEW commits
```

### Interactive rebase actions

|Action|Effect|
|---|---|
|`pick`|keep the commit as-is|
|`reword`|keep the change, edit the message|
|`edit`|stop here so you can amend the commit|
|`squash`|merge into the previous commit, combine both messages|
|`fixup`|merge into the previous commit, **discard** this message|
|`drop`|remove the commit entirely|
|`exec`|run a shell command after this commit (great for running tests at each step)|

Auto-squash workflow:

```bash
git commit --fixup=<sha>            # why: creates a commit marked "fixup! <original message>"
git rebase -i --autosquash HEAD~5   # why: automatically reorders and marks it as fixup
```

### Merge vs rebase — the classic interview question

||Merge|Rebase|
|---|---|---|
|History|True, non-linear, preserves context|Rewritten, linear, clean|
|SHAs|Unchanged|New SHA for every replayed commit|
|Merge commits|Yes (unless fast-forward)|None|
|Safety|Safe on shared branches|**Never on shared/pushed branches**|
|Conflicts|Resolved once|Potentially resolved once per commit|
|Traceability|Shows when a branch existed|Loses that a branch existed|
|Best for|Integrating a branch into main|Cleaning up your own branch before a PR|

**The golden rule of rebasing:** _Never rebase commits that exist outside your repository._

Why: rebasing makes new commits. Anyone who already has the old ones will have divergent history, and their next `git pull` creates duplicates and a mess.

**A solid answer to give:**

> "I default to merge for integration because the history stays truthful — I can see when a branch existed and what the actual state was, which matters during incident forensics. I use rebase only locally, to update my own feature branch off main before opening a PR, so the diff is clean. For the PR merge itself I prefer squash-merge into main: one commit per unit of work, which makes `git revert` trivial and keeps `git bisect` usable."

The tie-back to **revert and bisect** is what separates a senior answer from an aesthetic preference.

### git pull --rebase

```bash
git pull --rebase              # fetch, then replay your local commits on top of the remote's
git config --global pull.rebase true
```

Avoids noisy "Merge branch 'main' of origin" commits when you and a teammate both committed to the same branch. Safe because you're only rebasing commits you haven't pushed.

## 16. Cherry-pick

Apply the change introduced by one or more specific commits onto the current branch.

```bash
git cherry-pick <sha>              # apply one commit here (creates a NEW commit)
git cherry-pick <sha1> <sha2>      # multiple
git cherry-pick <sha1>..<sha2>     # a range, exclusive of sha1
git cherry-pick <sha1>^..<sha2>    # a range, inclusive of sha1
git cherry-pick -n <sha>           # apply to working dir/index without committing
git cherry-pick -x <sha>           # append "(cherry picked from commit ...)" — traceability
git cherry-pick --continue / --abort / --skip
```

Real-world DevOps use: backporting a security fix from `main` to `release/1.4` without pulling in unrelated features. Always use `-x` on release branches so the provenance is recorded.
