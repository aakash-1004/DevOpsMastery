---
title: Git Part 2 — Internals
type: reference
module: git
tags:
  - git
order: 2
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-01-Fundamentals]]"
next: "[[Git-Part-03-Branching-and-Merging]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 2 — Internals

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-01-Fundamentals|Git Part 1 — Fundamentals]] · → [[Git-Part-03-Branching-and-Merging|Git Part 3 — Branching & Merging]]
<!-- /nav -->

## 9. The .git directory

```
.git/
├── HEAD                 # ref: refs/heads/main  — what is currently checked out
├── config               # repo-local configuration
├── description          # used only by gitweb
├── index                # the staging area (binary)
├── hooks/               # client-side scripts (samples by default)
├── info/exclude         # local ignore rules
├── objects/             # ALL content: blobs, trees, commits, tags
│   ├── ab/cdef123...    # loose objects, sharded by first 2 hex chars
│   ├── info/
│   └── pack/            # packfiles: compressed, delta-encoded object bundles
├── refs/
│   ├── heads/           # local branches → each file holds a commit SHA
│   ├── tags/            # tags
│   └── remotes/origin/  # remote-tracking branches
├── packed-refs          # refs compacted into one file
└── logs/                # reflog — every movement of HEAD and refs
```

If you delete `.git/`, you delete the entire history — the working directory becomes ordinary files.

## 10. The four object types

Everything in Git is one of four object types, stored in `.git/objects`, keyed by the SHA of its content.

### Blob — file content

- Stores **raw bytes only**. No filename, no path, no permissions, no timestamp.
- Identical content anywhere in the repo → **one blob**, stored once (deduplication).
- Because blobs have no name, Git does not record renames. `git log --follow` _infers_ renames by similarity.

### Tree — a directory

A tree is a list of entries:

```
100644 blob a1b2c3...    README.md
100755 blob d4e5f6...    deploy.sh
040000 tree 789abc...    src
120000 blob 0f1e2d...    link-to-file
```

- One tree per directory; subdirectories are nested tree objects.
- **File mode lives in the tree, not the blob.** That's why `chmod +x` shows up as a change even though content is unchanged.
- Modes: `100644` normal file, `100755` executable, `120000` symlink, `040000` directory, `160000` gitlink (submodule).
- **Git cannot track empty directories** — a directory exists only if a tree entry references it. Hence the `.gitkeep` convention.

### Commit — a snapshot plus metadata

```
tree      9f2c8a1...             ← ONE root tree = full snapshot of the project
parent    4b7d3e9...             ← 0 parents = root commit, 1 = normal, 2+ = merge
author    Aakash <a@x.com> 1732... +0530
committer Aakash <a@x.com> 1732... +0530

fix: bump image tag
```

- A commit points to **exactly one root tree**, and through it reaches every file. So a commit is a **full snapshot**, not a diff.
- **Author vs committer**: author wrote the change, committer applied it. After rebase or cherry-pick, the author stays original and the committer becomes you.
- The commit SHA hashes tree + parents + author + committer + timestamps + message. Change _anything_ and you get a **different commit** — which is exactly why rebase/amend/filter-repo "rewrite" history by creating new objects and orphaning the old ones.

### Tag — an annotated pointer

- **Lightweight tag**: just a file in `.git/refs/tags/` holding a SHA. **No object created.**
- **Annotated tag**: a real object containing target SHA, tagger, date, message, optional GPG signature. This is the fourth object type.

Use annotated tags for releases — they carry who/when/why and can be signed.

### Object graph

```
tag ──▶ commit ──▶ tree ──┬──▶ blob (README.md)
            │             └──▶ tree (src/) ──▶ blob (main.py)
            └──▶ parent commit ──▶ ...
branch ref ──▶ commit
HEAD ──▶ branch ref
```

### Lab — prove it

```bash
mkdir /tmp/gitobj && cd /tmp/gitobj && git init   # why: clean throwaway repo
mkdir src
echo "hello" > README.md
echo "print('hi')" > src/main.py

git add .                                # why: creates BOTH blobs and writes them to .git/objects
find .git/objects -type f                # why: blobs exist BEFORE any commit — 'add' writes objects

git commit -m "first"
git cat-file -p HEAD                     # why: the commit object — tree SHA, author, message
git cat-file -p HEAD^{tree}              # why: root tree — name→SHA mapping
git cat-file -p HEAD:src                 # why: the subtree for src/ — proves trees nest
git cat-file -p HEAD:src/main.py         # why: the blob — content with no name attached

# Blobs are content-only:
echo "hello" > copy.md                   # why: identical content, different filename
git hash-object README.md copy.md        # why: SAME SHA twice — filename is irrelevant

# Trees hold permissions:
chmod +x src/main.py
git diff                                 # why: "old mode 100644 / new mode 100755"

# Rewriting creates new objects:
git log --oneline                        # why: note the SHA
git commit --amend -m "new message"
git log --oneline                        # why: DIFFERENT SHA — old commit orphaned, not edited
git reflog                               # why: the original SHA is still here and recoverable

# Inspect any object:
git cat-file -t <sha>                    # why: prints blob | tree | commit | tag
git cat-file -s <sha>                    # why: size in bytes
git rev-parse HEAD                       # why: resolve a ref to a full SHA
git count-objects -vH                    # why: loose vs packed object statistics
```

## 11. Refs, HEAD and revision syntax

**A branch is a file.**

```bash
cat .git/HEAD                 # ref: refs/heads/main
cat .git/refs/heads/main      # 9f2c8a1b...  ← the branch IS this 41-byte file
git rev-parse main            # same SHA, via porcelain
```

### Revision selectors — know these cold

|Syntax|Meaning|
|---|---|
|`HEAD`|Current commit|
|`HEAD~1`, `HEAD~`|One commit back, following the **first parent** each time|
|`HEAD~3`|Three generations back (first-parent path)|
|`HEAD^` , `HEAD^1`|**First parent** of HEAD|
|`HEAD^2`|**Second parent** — only meaningful on a merge commit|
|`HEAD^^`|Same as `HEAD~2`|
|`HEAD@{2}`|Where HEAD was 2 moves ago (**reflog**, local only)|
|`main@{yesterday}`|Where main pointed yesterday (reflog)|
|`main..feature`|Commits reachable from feature but **not** from main|
|`main...feature`|Symmetric difference — in either, not in both|
|`HEAD:path/file`|The blob for that file at that commit|
|`:/fix login`|Most recent commit whose message matches|
|`v1.2^{commit}`|Peel an annotated tag down to the commit it points at|

**`~` vs `^` — the classic trick question.** `~` walks up generations always taking the first parent. `^` selects _which parent_. They're identical on linear history, and differ only at merge commits.
