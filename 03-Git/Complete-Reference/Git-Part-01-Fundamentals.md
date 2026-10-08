---
title: Git Part 1 — Fundamentals
type: reference
module: git
tags:
  - git
order: 1
up: "[[Git-GitHub-Complete-Reference-MOC]]"
next: "[[Git-Part-02-Internals]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 1 — Fundamentals

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · → [[Git-Part-02-Internals|Git Part 2 — Internals]]
<!-- /nav -->

## 1. What is version control?

**Version control (VCS)** is a system that **records changes to files** **over time** so you can recall specific versions later, **see who changed what and when,** and **work in parallel without overwriting each other.**

Three generations:

| Type                   | Description                                                        | Example            | Problem                                                |
| ---------------------- | ------------------------------------------------------------------ | ------------------ | ------------------------------------------------------ |
| Local VCS              | Versions stored on one machine only                                | RCS                | No collaboration, single point of failure              |
| Centralised VCS (CVCS) | One central server holds all history; clients check out a snapshot | SVN, Perforce, CVS | Server down = nobody works. Server lost = history lost |
| Distributed VCS (DVCS) | Every clone is a full copy of the repo **including all history**   | **Git**, Mercurial | More concepts to learn, but no single point of failure |

## 2. What is Git?

**Git** is a free, open-source **distributed version control system** created by Linus Torvalds in 2005 to manage Linux kernel development.

Definition to use in an interview:

> "Git is a distributed version control system that **tracks changes to a set of files by storing full snapshots of the project, addressed by content hash**. Every clone contains the **complete history**, so operations like log, diff, branch and commit are local and instant."

**Key characteristics:**

- **Distributed** — every clone is a complete repository with full history. You can commit, branch, diff and view logs with no network.
- **Snapshot-based, not delta-based** — Git records the state of all files at each commit, not a list of changes. (Deltas are used internally at the storage layer for compression only.)
- **Content-addressable** — every object is named by the SHA-1 hash of its own content. Identical content is stored once.
- **Integrity-checked** — a hash covers the content, so silent corruption is detectable. History cannot be changed without changing every downstream hash.
- **Nearly everything is additive** — almost nothing is truly destroyed until garbage collection runs.
- **Cheap branching** — a branch is a 41-byte file containing a commit SHA.

> Note on hashing: Git historically uses SHA-1. Modern Git supports SHA-256 repositories (`git init --object-format=sha256`), but SHA-1 remains the default in practice. Git also added collision-detection hardening for SHA-1.

## 3. Git vs GitHub

This is asked in almost every junior/mid interview, and people still fumble it.

| Git                              | GitHub                                                             |
| -------------------------------- | ------------------------------------------------------------------ |
| A tool / CLI program             | A web platform / hosted service                                    |
| Runs locally on your machine     | Runs in the cloud (SaaS) or self-hosted (GitHub Enterprise Server) |
| Handles version control          | Handles hosting, collaboration, code review, CI/CD, issue tracking |
| Created by Linus Torvalds (2005) | Founded 2008, acquired by Microsoft in 2018                        |
| Works fully offline              | Requires network                                                   |
| Open source                      | Proprietary product                                                |

Alternatives to GitHub: **GitLab**, **Bitbucket**, **Azure DevOps Repos**, **AWS CodeCommit**, **Gitea** (self-hosted).

One-line answer: _"Git is the version control system. GitHub is a hosting platform built around Git that adds collaboration, review and automation on top of it."_

## 4. Installing and configuring Git

```bash
# Install
sudo apt update && sudo apt install -y git      # Debian/Ubuntu
sudo dnf install -y git                          # RHEL/Fedora
git --version                                    # verify

# Identity (required — goes into every commit object)
git config --global user.name  "Aakash Rao"
git config --global user.email "you@example.com"

# Useful defaults
git config --global init.defaultBranch main      # new repos start on main, not master
git config --global core.editor "vim"
git config --global pull.rebase false            # pull = fetch + merge (explicit)
git config --global push.default simple
git config --global core.autocrlf input          # Linux/macOS; use 'true' on Windows

# Inspect
git config --list --show-origin                  # every setting and which file it came from
git config user.email                            # single value
```

### Config scope hierarchy (narrower wins)

| Scope    | Flag                | File                   | Applies to               |
| -------- | ------------------- | ---------------------- | ------------------------ |
| System   | `--system`          | `/etc/gitconfig`       | All users on the machine |
| Global   | `--global`          | `~/.gitconfig`         | Current user, all repos  |
| Local    | `--local` (default) | `.git/config`          | Current repo only        |
| Worktree | `--worktree`        | `.git/config.worktree` | Single worktree          |

Precedence: **worktree > local > global > system**.

### Useful aliases

```bash
git config --global alias.st  status
git config --global alias.co  checkout
git config --global alias.br  branch
git config --global alias.lg  "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.unstage "restore --staged"
```

## 5. Core terminology

| Term                                 | Definition                                                                                                                                             |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Repository (repo)**                | A project tracked by Git. Contains your files plus a `.git/` directory holding all history and metadata.                                               |
| **Working directory / working tree** | The actual files you see and edit on disk.                                                                                                             |
| **Index / staging area**             | An intermediate area holding the exact content that will go into the next commit. Physically it's `.git/index`, a binary file.                         |
| **Commit**                           | An immutable snapshot of the whole project at a point in time, plus author, timestamp, message and parent link(s).                                     |
| **Branch**                           | A movable pointer to a commit. Literally a file under `.git/refs/heads/` containing a 40-character SHA.                                                |
| **HEAD**                             | A pointer to the commit you currently have checked out. Normally it points at a branch ref (`ref: refs/heads/main`), which in turn points at a commit. |
| **Ref**                              | A human-readable name pointing to a SHA. Branches, tags and remote-tracking branches are all refs.                                                     |
| **Remote**                           | A named URL of another copy of the repo. `origin` is the conventional default.                                                                         |
| **Remote-tracking branch**           | A local read-only pointer (`origin/main`) reflecting where that branch was on the remote at the last fetch.                                            |
| **Upstream / tracking branch**       | The remote branch your local branch is configured to push/pull against.                                                                                |
| **Clone**                            | A full local copy of a remote repository, including complete history.                                                                                  |
| **Fork**                             | A server-side copy of someone else's repository under your account (a GitHub concept, not a Git one).                                                  |
| **Merge**                            | Combining the histories of two branches.                                                                                                               |
| **Rebase**                           | Replaying commits from one branch onto another base, producing new commits with new SHAs.                                                              |
| **Conflict**                         | Two branches changed the same region of the same file; Git cannot decide automatically.                                                                |
| **Tag**                              | A fixed, non-moving pointer to a commit, used for releases.                                                                                            |
| **Stash**                            | A temporary shelf for uncommitted work.                                                                                                                |
| **Detached HEAD**                    | HEAD points directly at a commit rather than at a branch.                                                                                              |
| **Origin**                           | Conventional name of the remote you cloned from. Not a keyword — just a default name.                                                                  |
| **Fast-forward**                     | A merge where the target branch simply moves forward because its history is an ancestor of the source.                                                 |
| **Bare repository**                  | A repo with no working directory — only the `.git` contents. Used on servers. Created with `git init --bare`.                                          |

## 6. The three trees (the mental model everything depends on)

Git manages three "trees" (states) at all times:

```
  Working Directory  →  Index (Staging)  →  HEAD (Repository)
        (edit)              (git add)          (git commit)
```

|Tree|What it holds|How you change it|
|---|---|---|
|**Working directory**|Your real files on disk|Editing files|
|**Index / staging area**|The proposed next commit|`git add`, `git restore --staged`|
|**HEAD**|The last commit on the current branch|`git commit`, `git reset`|

Almost every "confusing" Git command becomes obvious once you ask: _which of the three trees does this move?_

- `git add` → working dir **to** index
- `git commit` → index **to** HEAD
- `git reset --soft` → moves HEAD only
- `git reset --mixed` (default) → moves HEAD + index
- `git reset --hard` → moves HEAD + index + working dir (destructive)
- `git checkout` / `git switch` → moves HEAD + index + working dir to another branch
- `git restore <file>` → index **to** working dir
- `git restore --staged <file>` → HEAD **to** index

### File lifecycle states

```
untracked → staged → committed → modified → staged → committed ...
```

- **Untracked** — Git has never seen this file.
- **Tracked** — Git knows about it. A tracked file is one of:
    - **Unmodified** — matches HEAD
    - **Modified** — differs from HEAD, not staged
    - **Staged** — change recorded in the index, ready to commit

## 7. Everyday commands

```bash
# Start a repo
git init                             # initialise a new repo in the current directory
git init --bare                      # server-side repo with no working tree
git clone <url>                      # copy a remote repo (creates 'origin' automatically)
git clone <url> <dir>                # clone into a specific directory
git clone -b <branch> <url>          # clone and check out a specific branch

# Inspect
git status                           # what's modified, staged, untracked
git status -s                        # short format (XY filename)
git diff                             # working dir vs index (unstaged changes)
git diff --staged                    # index vs HEAD (what will be committed)
git diff HEAD                        # working dir vs HEAD (everything)
git diff main..feature               # difference between two branches
git diff --name-only <a> <b>         # just the filenames (used heavily in CI)
git diff --stat                      # summary of insertions/deletions per file

# Stage
git add <file>
git add .                            # everything in current dir and below
git add -A                           # all changes in the whole repo, including deletions
git add -p                           # interactive: stage hunk by hunk (best practice)
git add -u                           # only already-tracked files

# Commit
git commit -m "message"
git commit -am "message"             # stage tracked files + commit (skips untracked)
git commit --amend                   # rewrite the last commit
git commit --amend --no-edit         # amend without changing the message
git commit --allow-empty -m "trigger" # empty commit, sometimes used to retrigger CI

# History
git log
git log --oneline --graph --decorate --all   # the one you should memorise
git log -p                           # show patches
git log --stat                       # show file change stats
git log -n 5                         # last 5 commits
git log --since="2 weeks ago"
git log --author="Aakash"
git log --grep="fix"                 # search commit messages
git log -S "functionName"            # "pickaxe": find commits that added/removed this string
git log -G "regex"                   # same but regex on the diff
git log --follow <file>              # history across renames
git log main..feature                # commits in feature not in main
git show <sha>                       # full details of one commit
git blame <file>                     # who last changed each line
git blame -L 10,20 <file>            # only lines 10–20
git shortlog -sn                     # commit count per author

# Remove / move
git rm <file>                        # delete and stage the deletion
git rm --cached <file>               # untrack but keep the file on disk
git mv old new                       # rename (equivalent to mv + add + rm)
```

### Commit message convention (Conventional Commits)

Widely used, and it powers automated changelogs and semantic-release tooling.

```
<type>(<optional scope>): <short imperative summary>

<optional body — why, not what>

<optional footer — BREAKING CHANGE:, Refs: JIRA-123>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`.

Examples:

```
feat(api): add pagination to /users endpoint
fix(pipeline): pin kubectl to v1.29 to stop drift
ci: cache node_modules between GitHub Actions runs
```

Rules: imperative mood ("add", not "added"), subject under 50 chars, no trailing period, body wrapped at 72.

## 8. .gitignore

Tells Git which **untracked** files to ignore.

```gitignore
# Comments start with #

*.log                  # any .log file, anywhere
build/                 # directory named build, anywhere
/dist                  # dist only at repo root
!important.log         # negation — do NOT ignore this one
temp?.txt              # ? = single character
**/node_modules        # ** = any number of directories
docs/**/*.pdf
```

**Critical gotcha (very common interview question):** `.gitignore` only affects **untracked** files. If a file is already tracked, adding it to `.gitignore` does nothing.

```bash
git rm --cached secrets.env        # untrack it (keeps it on disk)
echo "secrets.env" >> .gitignore
git commit -m "stop tracking secrets.env"
```

Debug why a file is (or isn't) ignored:

```bash
git check-ignore -v <file>         # prints which .gitignore line matched
git status --ignored               # list ignored files
```

Other ignore mechanisms:

- `.git/info/exclude` — local-only ignores, not committed, personal to you
- `core.excludesFile` — global ignore file (`git config --global core.excludesFile ~/.gitignore_global`)
- `.gitattributes` — different purpose: line endings, diff drivers, LFS, `export-ignore`, merge strategies

```gitattributes
* text=auto                         # normalise line endings
*.sh text eol=lf                    # force LF on shell scripts
*.png binary                        # don't try to diff
*.tfstate merge=ours                # custom merge strategy
terraform.lock.hcl linguist-generated=true
```
