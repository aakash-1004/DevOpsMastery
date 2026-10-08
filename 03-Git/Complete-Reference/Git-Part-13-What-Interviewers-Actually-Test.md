---
title: Git Part 13 — What Interviewers Actually Test
type: reference
module: git
tags:
  - git
order: 13
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-12-Cheat-Sheet]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 13 — What Interviewers Actually Test

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-12-Cheat-Sheet|Git Part 12 — Cheat Sheet]]
<!-- /nav -->

Ranked by how often it appears:

1. **`reset` vs `revert`** — near-universal. Know the three reset modes and the "safe on shared branches" rule.
2. **`merge` vs `rebase`** — near-universal. Answer with the golden rule plus revert/bisect reasoning.
3. **Resolving merge conflicts** — they want the workflow, not theory.
4. **`fetch` vs `pull`** — quick screening question.
5. **Secrets committed to the repo** — the scenario question. Lead with _rotate_.
6. **Branching strategy** — expect "which do you use and why". Have a concrete answer.
7. **Reflog recovery** — the "I lost my work" scenario.
8. **Git internals (objects)** — asked at senior level to check depth beyond command memorisation.
9. **GitOps / Git in CI/CD** — DevOps-specific. Tie rollback to `git revert`.
10. **Branch protection / CODEOWNERS / signed commits** — senior process and supply-chain questions.
11. **`git bisect`** — rarely practised, high-impact answer.
12. **Monorepo CI patterns** — sparse checkout, path filters, changed-file detection.

## Phrases that read as senior

- "`--force-with-lease`, never plain `--force`."
- "Rotate the credential first — removing it from history doesn't un-leak it."
- "I squash-merge so `git revert` and `git bisect` stay usable."
- "Local hooks are a convenience; enforcement lives in branch protection and CI."
- "A commit SHA is a fixed snapshot of desired state — that's why GitOps works."
- "Pin third-party actions to a SHA; tags are mutable."
- "Nothing is lost until `gc --prune` runs."

## Red flags to avoid

- Saying Git stores diffs.
- Saying `git pull` is always safe.
- Recommending `git filter-branch` (deprecated).
- Suggesting `reset --hard` on a shared branch to undo a bad push.
- Claiming client-side hooks enforce policy.
- Saying you'd fix a leaked secret by only rewriting history.

---

## Related notes

- [[5.1-Kubernetes-Core-Concepts|Kubernetes — Core Concepts]]
- [[7.1-GitHub-Actions|GitHub Actions — CI/CD Pipelines]]
- [[7.3-ArgoCD-and-GitOps|ArgoCD and GitOps]]
- [[08-State-Management-and-Remote-Backend|Terraform — State and Drift]]
- [[7.4-Jenkins-CICD-Pipeline|Jenkins — Pipelines and Agents]]
- [[4.3-Docker-Images-and-Containers|Docker — Images and Layers]]
- [[Interview-Prep-MOC|DevOps Interview Question Bank]]
- [[1.4-Linux-Performance-Troubleshooting|Linux Troubleshooting]]

## Practice checklist

- [ ] Build the object-model lab and inspect blob/tree/commit with `cat-file`
- [ ] Deliberately break a repo with `reset --hard` and recover it via `reflog`
- [ ] Run `git bisect run` against a seeded regression
- [ ] Rewrite history with `git filter-repo` on a throwaway repo
- [ ] Enable `rerere` and rebase through the same conflict twice
- [ ] Set up `pre-commit` with `gitleaks` on a real repo
- [ ] Configure branch protection + CODEOWNERS on a personal repo
- [ ] Wire GitHub Actions → AWS via OIDC (no stored keys)
- [ ] Practise saying answers 21, 23, 43, 50 and 60 out loud
