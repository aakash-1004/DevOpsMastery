---
title: Git Part 11 — Interview Question Bank
type: reference
module: git
tags:
  - git
order: 11
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-10-Advanced-Internals-and-Performance]]"
next: "[[Git-Part-12-Cheat-Sheet]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 11 — Interview Question Bank

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-10-Advanced-Internals-and-Performance|Git Part 10 — Advanced Internals & Performance]] · → [[Git-Part-12-Cheat-Sheet|Git Part 12 — Cheat Sheet]]
<!-- /nav -->

Answers are written the way you should _say_ them — short, direct, then one supporting detail.

## A. Basic (screening / fresher level)

**1. What is Git?** A distributed version control system that tracks changes by storing full snapshots of a project, addressed by content hash. Every clone has the complete history, so commit, branch, diff and log are local and instant.

**2. Git vs GitHub?** Git is the version control tool that runs locally. GitHub is a hosting platform built around Git that adds collaboration, code review, issue tracking and CI/CD.

**3. Centralised vs distributed VCS?** In centralised VCS (SVN) there's one server with the history and clients hold a working copy — server down means nobody works. In distributed VCS every clone is a full repository with complete history, so you can work offline and there's no single point of failure.

**4. What is a repository?** A project tracked by Git. It's your files plus a `.git` directory containing all objects, refs and metadata.

**5. What is a commit?** An immutable snapshot of the entire project at a point in time, plus author, committer, timestamp, message and parent commit link(s). It's identified by the SHA of its own content.

**6. What is a branch?** A movable pointer to a commit. Physically it's a file under `.git/refs/heads/` containing a 40-character SHA — which is why creating a branch is instant.

**7. What is HEAD?** A pointer to what you currently have checked out. Normally it points at a branch ref, which points at a commit. If it points directly at a commit, that's detached HEAD.

**8. What is the staging area?** An intermediate area (`.git/index`) holding exactly what will go into the next commit. It lets you commit a subset of your changes instead of everything you've edited.

**9. `git add` vs `git commit`?** `add` moves changes from the working directory into the staging area. `commit` takes what's staged and records it as a new commit.

**10. `git fetch` vs `git pull`?** `fetch` downloads objects and updates remote-tracking refs — it never touches your working branch, so it's always safe. `pull` is `fetch` plus an automatic merge or rebase into your current branch.

**11. `git clone` vs `git pull`?** `clone` creates a new local copy of a repo that doesn't exist yet. `pull` updates an existing local repo.

**12. What is `origin`?** The conventional default name for the remote you cloned from. It's just a name, not a keyword.

**13. What is a merge conflict and how do you resolve it?** It happens when both branches changed the same region of the same file. Git marks the conflicted region, you edit the file to the correct content, `git add` it to mark it resolved, then commit. `git merge --abort` backs out entirely.

**14. What does `.gitignore` do?** It tells Git which untracked files to ignore. It has no effect on files that are already tracked — for those you need `git rm --cached` first.

**15. `git rm` vs `git rm --cached`?** `git rm` deletes the file and stages the deletion. `--cached` untracks it but leaves it on disk.

**16. How do you undo the last commit?** Depends on intent: `git reset --soft HEAD~1` keeps the changes staged, `--mixed` keeps them unstaged, `--hard` discards them. If the commit is already pushed, use `git revert` instead.

**17. How do you change the last commit message?** `git commit --amend`. It creates a new commit object with a new SHA, so only do it before pushing.

**18. What is a tag?** A fixed pointer to a commit, used for releases. Lightweight tags are just a ref; annotated tags are real objects carrying tagger, date, message and an optional signature. Use annotated for releases.

**19. What is `git stash`?** A temporary shelf for uncommitted work so you can switch branches. `git stash push -m "msg"` to save, `git stash pop` to restore.

**20. How do you see the history?** `git log`. In practice `git log --oneline --graph --decorate --all` for a readable branch graph.

## B. Intermediate

**21. `reset` vs `revert`?** `reset` moves the branch pointer backwards and rewrites history — local branches only. `revert` creates a new commit that inverts a previous one, preserving history — the only safe undo on a shared branch.

**22. Explain the three reset modes.** `--soft` moves HEAD only, changes stay staged. `--mixed` (default) moves HEAD and resets the index, changes stay in the working directory. `--hard` resets all three trees and discards the work.

**23. Merge vs rebase?** Merge preserves true history and creates a merge commit; it's safe on shared branches. Rebase replays your commits onto a new base, producing new SHAs and a linear history; never do it on commits that exist outside your repo. I use rebase locally to update my feature branch, and squash-merge for the PR itself.

**24. Why is rebasing a shared branch dangerous?** Rebase creates new commits with new SHAs. Anyone who already pulled the old commits now has divergent history — their next pull produces duplicates and conflicts, and someone will eventually force-push over someone else's work.

**25. What is a fast-forward merge?** When the target branch's tip is an ancestor of the source, Git just moves the pointer forward — no merge commit. `--no-ff` forces a merge commit so you can still see that a branch existed.

**26. What is `git cherry-pick` and when do you use it?** It applies the change from a specific commit onto the current branch as a new commit. I use it to backport a hotfix from `main` to a release branch, with `-x` so the origin commit is recorded in the message.

**27. What is `git reflog` and why does it matter?** A local log of every position HEAD and each branch has held. It's how you recover from a bad `reset --hard`, a botched rebase, or a deleted branch. It's local-only and expires after about 90 days.

**28. A colleague deleted a branch with unmerged work. Can you recover it?** Yes, if it's still on their machine or yours. Find the tip SHA in `git reflog`, then `git branch recovery <sha>`. Objects survive until garbage collection runs, so `git fsck --lost-found` also finds dangling commits.

**29. `HEAD~2` vs `HEAD^2`?** `~` walks up generations always taking the first parent. `^` selects which parent — `HEAD^2` is the second parent, which only exists on a merge commit. They're identical on linear history.

**30. `git diff` vs `git diff --staged`?** `git diff` compares the working directory to the index — your unstaged changes. `--staged` compares the index to HEAD — what will actually be committed.

**31. How do you squash commits?** `git rebase -i HEAD~n` and mark commits as `squash` or `fixup`. Or just use GitHub's "Squash and merge" on the PR, which is what I'd normally do.

**32. What is detached HEAD and how do you get out?** HEAD points directly at a commit instead of a branch. Commits made there belong to no branch and get orphaned when you switch away. Create a branch at that point with `git switch -c <name>` to keep the work.

**33. `--force` vs `--force-with-lease`?** `--force` overwrites the remote unconditionally and can destroy a colleague's pushed work. `--force-with-lease` checks the remote is still where you last saw it and refuses if someone else pushed. Always use the lease version.

**34. How do you find which commit introduced a bug?** `git bisect`. Mark a known-good commit and a known-bad one; Git binary-searches the range. With `git bisect run ./test.sh` it's fully automated — about ten tests for a thousand commits.

**35. What does `git clean -fdx` do?** Deletes untracked files (`-f`), untracked directories (`-d`) and ignored files (`-x`). It's the standard CI workspace reset. Always dry-run with `-nd` first.

**36. What are Git hooks? Client-side vs server-side?** Scripts that fire at lifecycle points. Client-side (`pre-commit`, `commit-msg`, `pre-push`) run on the developer's machine, aren't versioned, and are bypassable with `--no-verify`. Server-side (`pre-receive`, `update`, `post-receive`) run on the server and can actually reject a push. Real enforcement belongs in CI and branch protection, not client hooks.

**37. How do you make hooks shareable across a team?** Commit them to a directory like `.githooks/` and set `core.hooksPath`, or use the `pre-commit` framework with a committed `.pre-commit-config.yaml`.

**38. What is a bare repository?** A repo with no working directory — only the `.git` contents. It's what you put on a server, because you can't push to a checked-out branch of a non-bare repo.

**39. Submodule vs subtree?** A submodule is a pointer to a specific commit in another repo — precise pinning, but awkward UX and detached HEADs. A subtree vendors the files into your repo — simpler for consumers, messier history. For shared code I'd avoid both and publish a versioned artifact instead.

**40. How do you handle a large binary file in Git?** Git LFS stores a pointer in Git and the bytes on a separate server. Better still, keep build artifacts out of Git entirely and use S3, Artifactory or a container registry.

**41. What is `git worktree`?** It checks out multiple branches into separate directories sharing one `.git`. Useful for a hotfix without stashing your current work, or building two refs side by side in CI.

**42. What is `git blame` used for?** Showing who last modified each line of a file and in which commit. It's for finding context and the right person to ask, not for assigning blame.

## C. Advanced / senior

**43. How does Git store data internally?** As a content-addressable object database. Four object types: blobs hold raw file bytes with no name; trees map names and file modes to blobs and other trees, forming directories; commits point to exactly one root tree plus parent commits and metadata; annotated tags point to a commit with tagger info. Every object is keyed by the SHA of its own content, so identical content is stored once.

**44. Does Git store diffs or snapshots?** Snapshots. Each commit references a complete tree of the project. Deltas exist only inside packfiles as a storage-level compression optimisation — the data model itself is snapshot-based. `git diff` computes differences on demand.

**45. Why is a commit hash immutable?** The SHA is computed over the tree, the parent SHAs, author, committer, timestamps and message. Change any of those and you get a different hash — which means a different commit, not a modified one. That's exactly why rebase, amend and filter-repo create new objects and orphan the old ones.

**46. Why do rename operations not appear in Git's data model?** Blobs store content without filenames; names live in trees. A rename is a tree change with the same blob SHA. Git infers renames by similarity at diff time — that's what `git log --follow` and `-M` do.

**47. Why does `chmod +x` show as a change?** Because file mode is stored in the tree entry, not the blob. The content hash is unchanged but the tree entry goes from `100644` to `100755`.

**48. Why can't Git track empty directories?** A directory only exists as an entry in a tree object, and a tree entry must point to something. No files, no tree, no directory. Hence the `.gitkeep` convention.

**49. Explain packfiles and garbage collection.** Loose objects are one file each, which doesn't scale. `git gc` packs them into packfiles with delta compression between similar objects, and prunes unreachable objects past the reflog expiry. Nothing you "lose" is actually gone until `gc --prune` runs.

**50. Someone committed AWS credentials. Walk me through it.** First, rotate the credential — immediately. The commit may already be cloned, mirrored or scraped, and removing it from history doesn't un-leak it. Then strip it from history with `git filter-repo --invert-paths --path <file>` or `--replace-text`, force-push all branches and tags, and have everyone re-clone. Ask GitHub support to purge the cached commit views, and check CloudTrail for any use of the key. Finally, prevent recurrence: gitleaks in pre-commit, GitHub secret scanning with push protection, and move the secret into Secrets Manager or Vault.

**51. How do you enforce code quality and process on a repo?** Branch protection on `main`: require a PR, require N approvals, require Code Owner review, require named status checks to pass, require branches be up to date, dismiss stale reviews, block force pushes and deletions, and include administrators. Path-based ownership via `CODEOWNERS`. Signed commits if the org requires provenance. Client-side hooks as a convenience on top, never as the enforcement layer.

**52. What are signed commits and why would you require them?** `git commit -S` signs the commit with a GPG or SSH key, proving who authored it. Without signing, `user.email` is just a string anyone can set. Required in supply-chain-sensitive environments so you can prove a commit came from a trusted identity.

**53. How do you handle a monorepo in CI?** Path filters so only affected pipelines run, `git diff --name-only origin/main...HEAD` to compute the change set, sparse-checkout to materialise only the needed paths, and partial/blobless clone to keep checkout fast. Plus a merge queue so PRs are tested against the latest trunk.

**54. What's the difference between `main..feature` and `main...feature`?** Two dots gives commits reachable from `feature` but not `main` — "what's new on the branch". Three dots gives the symmetric difference — commits in either but not both. In `git diff`, three dots means "diff against the merge base", which is what PR diffs show.

**55. What is `rerere`?** REuse REcorded REsolution. Git records how you resolved a conflict and replays it automatically the next time the same conflict appears. It's invaluable on long-running branches where you repeatedly rebase and hit the same conflict.

**56. How do you revert a merge commit?** `git revert -m 1 <merge-sha>` — `-m 1` tells Git which parent is the mainline. The gotcha is that once you've reverted a merge, re-merging that branch later won't bring the changes back, because Git considers them already merged. You have to revert the revert, or rebase the branch.

**57. What happens to the commits on a branch you delete?** Nothing immediately. The objects still exist; they're just unreachable. The branch's last position is in the reflog, and `git fsck --lost-found` finds dangling commits. They're only destroyed when `git gc --prune` runs past the expiry window.

**58. Explain shallow vs partial clone.** Shallow (`--depth N`) truncates history to the last N commits — fast, but breaks `git describe`, `log --follow` and blame-based tools. Partial clone (`--filter=blob:none`) keeps the full commit graph but downloads file contents lazily on demand — better for CI when you still need history.

**59. How would you migrate a large SVN/Perforce repo to Git?** `git svn clone` with an authors map for SVN, or a purpose-built tool for Perforce. Then clean up: strip large binaries into LFS or an artifact store with `filter-repo`, rewrite tags, verify the commit count and file tree against the source, and run `gc --aggressive`. Cut over with a freeze window and a read-only period on the old system.

**60. What's your branching strategy and why?** For a containerised service deploying multiple times a day: trunk-based with short-lived branches, feature flags for incomplete work, squash-merge into `main`, and branch protection with required checks. GitFlow only if we ship versioned artifacts to customers and support multiple releases in the field simultaneously — otherwise it's overhead that slows delivery.

## D. GitHub-specific

**61. What is a fork and when do you use it instead of a branch?** A fork is a server-side copy of a repo under your own account. You fork when you don't have write access — typically open-source contribution. Within a team with write access, branches are simpler.

**62. Explain GitHub's three merge options.** Merge commit keeps every commit plus adds a merge commit. Squash and merge collapses the PR into one commit on main — my default, because it makes revert and bisect clean. Rebase and merge replays commits linearly with no merge commit.

**63. What is CODEOWNERS?** A file that maps paths to owning teams, so changes to those paths automatically request review from the right people. Combined with "require Code Owner review", it means nobody can merge a Terraform change without the infra team's approval.

**64. What's a merge queue and why does it matter?** It serialises merges and re-tests each PR against the latest main. It catches semantic conflicts — two PRs that pass individually but break when combined. Important once you have enough PR throughput that main can break between CI runs.

**65. What is `GITHUB_TOKEN` and how do you secure it?** An automatically generated token scoped to the repository, created per workflow run and revoked when it ends. Secure it by setting `permissions:` explicitly to least privilege at the workflow or job level rather than relying on the default.

**66. How do you authenticate GitHub Actions to AWS without storing keys?** OIDC. Set `permissions: id-token: write`, configure an IAM OIDC identity provider for GitHub, and use `aws-actions/configure-aws-credentials` with `role-to-assume`. The workflow gets short-lived STS credentials. No long-lived access keys in secrets at all.

**67. Difference between `pull_request` and `pull_request_target`?** `pull_request` runs in the context of the PR's merge commit with no access to repo secrets for fork PRs — safe. `pull_request_target` runs with the base branch's workflow _and_ repo secrets. If you check out PR code inside it, untrusted code runs with your secrets. It's a well-known security footgun.

**68. Why pin GitHub Actions to a SHA instead of a tag?** Tags are mutable — an action author (or an attacker with access) can move `v3` to point at different code. Pinning to a full commit SHA makes the dependency immutable. It's a standard supply-chain control.

**69. What's the difference between a deploy key, a PAT, and a GitHub App?** A deploy key is an SSH key scoped to one repository, read-only by default — best for CI clone access. A PAT is tied to a user account and inherits that user's access, which is over-broad and breaks when the person leaves. A GitHub App has its own identity with fine-grained per-repo permissions and short-lived tokens — the right choice for automation.

**70. How do you protect against secrets being pushed?** GitHub secret scanning with push protection blocks the push at the server. On top of that, gitleaks or trufflehog in a pre-commit hook and as a required CI check.

## E. Scenario-based

**71. You pushed a commit to `main` that broke production. What do you do?** Revert first, investigate second. `git revert <sha>` creates an inverse commit, push it, and let the pipeline redeploy. I don't reset `main` — that rewrites shared history and breaks everyone's clones. Once production is stable, I find the root cause and fix forward.

**72. You committed to the wrong branch. How do you move it?** If it's not pushed: note the SHA, `git reset --hard HEAD~1` on the wrong branch, switch to the correct branch and `git cherry-pick <sha>`. If it's already pushed to a shared branch, revert it there and cherry-pick to the right one.

**73. Your feature branch is 40 commits behind main and conflicts badly. What now?** If the branch is mine and unshared, `git rebase main` and resolve conflicts, with `rerere` enabled so repeated conflicts resolve themselves. If it's shared, merge `main` into it instead. If it's genuinely 40 commits behind, that's a process problem — I'd also raise that branches should be merged within a day or two.

**74. Two engineers force-pushed over each other. How do you recover?** Find the lost tip SHA in the reflog of whoever still has it locally, or via the GitHub Events API which records every push including overwritten ones. Create a recovery branch at that SHA and merge. Then turn on branch protection to block force pushes, which is the actual fix.

**75. CI is slow because checkout takes 4 minutes on a huge repo. What do you change?** Use a blobless partial clone (`--filter=blob:none`) rather than a shallow clone if history is needed, sparse-checkout to only the paths the job touches, cache the repo on self-hosted runners, and enable `git maintenance` / commit-graph. If it's a monorepo, add path filters so most jobs don't run at all.

**76. How do you roll back a Kubernetes deployment managed by ArgoCD?** `git revert` the config commit in the manifests repo and push. ArgoCD detects the new commit and reconciles the cluster back. The rollback is itself a reviewed, auditable commit — that's the point of GitOps. `kubectl rollout undo` would work too, but ArgoCD would detect the drift and sync it straight back to what Git says.

**77. A developer says "it works on my machine but CI fails". Where does Git come in?** Usually something isn't committed, or is gitignored, or has a different line ending. I'd check `git status` for untracked files, `git check-ignore -v` on the suspect path, and `.gitattributes` for CRLF handling. Also file mode — a script that's executable locally but committed as `100644` fails in CI.

**78. How would you audit who deployed what to production last quarter?** Look at the Git history of the config repo — every deployment is a commit with an author, timestamp and a linked PR with approvals. That's the strongest argument for GitOps: the deployment log and the audit log are the same artifact.

**79. You need to remove a 2 GB file that was committed a year ago and is bloating every clone.** `git filter-repo --strip-blobs-bigger-than 100M` or target the specific path, then force-push and have everyone re-clone. Before that, confirm the file isn't needed, move it to S3 or an artifact registry, and schedule it — it's a breaking change for every open PR and every collaborator.

**80. How do you prevent direct pushes to main?** Branch protection with "require a pull request before merging", block force pushes and deletions, and include administrators. Server-side. A `pre-push` hook is not sufficient because it's bypassable and only exists on machines where someone installed it.
