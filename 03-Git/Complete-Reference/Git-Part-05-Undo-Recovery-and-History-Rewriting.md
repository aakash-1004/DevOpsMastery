---
title: "Git Part 5 — Undo, Recovery & History Rewriting"
type: reference
module: git
tags:
  - git
order: 5
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-04-Remotes-and-Collaboration]]"
next: "[[Git-Part-06-Tags-Submodules-Hooks-Worktrees-LFS]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 5 — Undo, Recovery & History Rewriting

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-04-Remotes-and-Collaboration|Git Part 4 — Remotes & Collaboration]] · → [[Git-Part-06-Tags-Submodules-Hooks-Worktrees-LFS|Git Part 6 — Tags, Submodules, Hooks, Worktrees, LFS]]
<!-- /nav -->

This is the heaviest interview area. If you can explain `reset` vs `revert`, and recover with `reflog`, you have cleared most Git questions.

## 21. reset, revert, restore, checkout

```bash
# reset — MOVES the branch pointer. Rewrites history. Local use only.
git reset --soft  HEAD~1     # why: undo the commit, keep changes STAGED (fix a message, squash)
git reset --mixed HEAD~1     # why: DEFAULT. Undo commit, keep changes in working dir, unstaged
git reset --hard  HEAD~1     # why: undo commit AND discard all changes — destructive
git reset <sha>              # why: move current branch to any commit
git reset --hard origin/main # why: force local branch to exactly match the remote

# revert — ADDS a new commit that inverts a previous one. Safe on shared branches.
git revert <sha>
git revert --no-commit <sha> # why: stage the inverse without committing yet
git revert -m 1 <merge-sha>  # why: revert a MERGE — -m 1 says "keep the first parent as mainline"
git revert <sha1>..<sha2>    # why: revert a range

# restore — modern, file-level, non-history operations
git restore <file>                 # why: discard working-dir changes (was: checkout -- <file>)
git restore --staged <file>        # why: unstage without touching the working dir
git restore --source=HEAD~2 <file> # why: get a file as it was two commits ago
git restore .                      # why: discard ALL working-dir changes
```

### The three reset modes against the three trees

|Mode|HEAD|Index|Working dir|Use when|
|---|---|---|---|---|
|`--soft`|moved|unchanged|unchanged|Re-commit with a better message, or squash locally|
|`--mixed`|moved|reset|unchanged|Unstage everything, restructure the commit|
|`--hard`|moved|reset|reset|Throw the work away entirely|

### reset vs revert — the rule

||`reset`|`revert`|
|---|---|---|
|Effect|Moves the branch pointer backwards|Creates a new inverse commit|
|History|Rewritten|Preserved, appended to|
|Shared branches|**Never**|**Yes — this is the only safe undo on `main`**|
|Audit trail|Lost|Kept|

One-liner: _"`reset` rewrites history, so it's for local branches only. `revert` adds history, so it's the only safe way to undo something already pushed to a shared branch."_

**Reverting a merge commit:** a merge has two parents, so Git doesn't know which line of history is "mainline." `-m 1` = first parent (usually main). Gotcha: after reverting a merge, re-merging that branch later won't bring the changes back — you must revert the revert, or rebase the branch.

## 22. reflog — the safety net

The **reflog** is a local log of every position `HEAD` (and each branch) has held. It is how you recover from a bad `reset`, a botched `rebase`, or a deleted branch.

```bash
git reflog                       # why: every HEAD movement with a HEAD@{n} index
git reflog show main             # why: reflog for a specific branch
git reset --hard HEAD@{3}        # why: jump back to a pre-disaster state
git branch recovery <sha>        # why: resurrect an orphaned commit under a new branch name
git checkout -b rescue HEAD@{5}
```

Key facts:

- **Local only.** Never pushed, never cloned. A fresh clone has no reflog.
- Default expiry: **90 days** for reachable entries, **30 days** for unreachable ones.
- Nothing is actually deleted until garbage collection runs.

```bash
git fsck --lost-found            # why: find dangling commits/blobs not in the reflog either
git gc                           # why: pack loose objects and prune unreachable ones
git gc --prune=now --aggressive  # why: force cleanup (this is what actually destroys orphans)
```

**The canonical answer to "I hard-reset and lost a commit":**

> "Nothing is lost yet. Git objects are immutable and unreachable objects survive until GC. I'd run `git reflog`, find the SHA from before the reset, and either `git reset --hard <sha>` or create a recovery branch at that SHA. If it predates the reflog, `git fsck --lost-found` finds dangling commits."

## 23. Stash

```bash
git stash                            # shelve tracked modified files
git stash push -m "wip: refactor"    # with a message (preferred)
git stash -u                         # include untracked files
git stash -a                         # include ignored files too
git stash push -- path/to/file       # stash only specific paths
git stash list                       # stash@{0}, stash@{1}, ...
git stash show -p stash@{1}          # view the diff
git stash pop                        # apply the top stash and remove it
git stash apply stash@{2}            # apply but keep it in the list
git stash drop stash@{0}
git stash clear                      # delete all stashes
git stash branch newbranch           # create a branch from a stash (when the base has moved on)
```

Stashes are stored as commits under `refs/stash`, so a dropped stash is recoverable via `git fsck --unreachable`.

## 24. Rewriting history

All of these **create new commits with new SHAs**. Only use them on commits that have not been shared, or on your own PR branch where you control the force-push.

```bash
git commit --amend                    # rewrite the last commit (message and/or content)
git rebase -i HEAD~n                  # squash, reword, drop, reorder
git filter-repo --path secrets.env --invert-paths    # remove a path from ALL history
git filter-repo --replace-text expressions.txt       # redact strings across all history
```

- `git filter-branch` is **deprecated** — slow and full of footguns. Use **`git filter-repo`** (recommended by the Git project) or **BFG Repo-Cleaner**.
- After rewriting shared history, every collaborator must re-clone or hard-reset. Open PRs will break.

## 25. Secrets committed to history — the scenario question

Very common, and directly relevant to DevOps. The order matters more than the commands.

1. **Rotate the credential immediately.** This is step one, always. The commit may already be cloned, mirrored, cached by the host, or scraped by a bot. Cleaning history does **not** un-leak a secret.
2. **Remove it from history:**

    ```bash
    pip install git-filter-repogit filter-repo --path config/secrets.env --invert-paths   # why: strip the file from every commit# or redact just the string:git filter-repo --replace-text expressions.txt             # file contains: AKIA...==>REMOVED
    ```

3. **Force-push all branches and tags**, then have every collaborator re-clone.
4. **Ask the host to purge cached views** — on GitHub, old commit SHAs remain reachable via the API until support garbage-collects them.
5. **Audit for use** — check CloudTrail / provider logs for any use of the leaked credential.
6. **Prevent recurrence:**
    - Pre-commit scanning: `gitleaks`, `trufflehog`, `detect-secrets`
    - GitHub **secret scanning** + **push protection**
    - A secrets manager: AWS Secrets Manager, HashiCorp Vault, SOPS, Sealed Secrets / External Secrets Operator for Kubernetes
    - `.gitignore` the file _and_ verify with `git check-ignore -v`

**Leading with "rotate first" is what separates a senior answer from a junior one.**

## 26. git bisect — find the commit that broke it

Binary search across history. Strong senior signal; almost nobody practises it.

```bash
git bisect start
git bisect bad                 # why: current commit is broken
git bisect good v1.2.0         # why: last known-good commit or tag
# Git checks out a midpoint. Test it, then:
git bisect good   # or  git bisect bad
# repeat until Git prints "<sha> is the first bad commit"
git bisect reset               # why: return to your original HEAD

# Fully automated:
git bisect start HEAD v1.2.0
git bisect run ./test.sh       # why: exit 0 = good, 1-124 = bad, 125 = skip
```

Finds the culprit in ~log2(N) steps — 10 tests across 1000 commits. This is also _why_ squash-merging matters: bisect over clean, individually-buildable commits actually works.

## 27. Other recovery and inspection tools

```bash
git fsck --full --unreachable     # why: list all dangling objects
git show <dangling-sha>           # why: inspect a recovered object
git cherry -v main feature        # why: which feature commits are NOT in main
git rev-list --count main         # why: number of commits
git describe --tags               # why: v1.4.2-3-gabc1234 — nearest tag + distance + SHA
git archive -o rel.tar.gz HEAD    # why: export a tree without .git (for packaging)
git clean -nd                     # why: DRY RUN — show what would be deleted
git clean -fd                     # why: delete untracked files and directories
git clean -fdx                    # why: also delete ignored files — standard CI workspace reset
```
