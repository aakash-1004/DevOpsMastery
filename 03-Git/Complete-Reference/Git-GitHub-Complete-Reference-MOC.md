---
title: "Git & GitHub — Complete Reference"
aliases:
  - "3.0-Git & GitHub — Complete Reference"
  - 3.0-Git-GitHub-Complete-Reference
type: moc
module: git
tags:
  - moc
  - git
up: "[[Git-MOC]]"
related:
  - "[[3.1-Git-Workflow]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git & GitHub — Complete Reference

<!-- nav -->
↑ [[Git-MOC|Git & GitHub]]
<!-- /nav -->

> Scope: everything a DevOps / Cloud engineer with 3+ years experience is expected to know, from first principles to senior-level internals, plus a full interview question bank at the end.

**How to use this note**

- Sections 1–4 = fundamentals. Read once, then use as lookup.
- Sections 5–11 = the working knowledge you use daily.
- Sections 12–16 = senior differentiators. This is where interviews are won or lost.
- Section 17 = interview question bank (basic → scenario-based).
- Section 18 = command cheat sheet.

## Contents

- [[Git-Part-01-Fundamentals|Git Part 1 — Fundamentals]] — What is version control?; What is Git?; Git vs GitHub; Installing and configuring Git; Core terminology; The three trees (the mental model everything depends on)
- [[Git-Part-02-Internals|Git Part 2 — Internals]] — The .git directory; The four object types; Refs, HEAD and revision syntax
- [[Git-Part-03-Branching-and-Merging|Git Part 3 — Branching & Merging]] — Branches; Merging; Conflicts; Rebase; Cherry-pick
- [[Git-Part-04-Remotes-and-Collaboration|Git Part 4 — Remotes & Collaboration]] — Remotes; fetch, pull, push; Tracking branches; SSH vs HTTPS authentication
- [[Git-Part-05-Undo-Recovery-and-History-Rewriting|Git Part 5 — Undo, Recovery & History Rewriting]] — reset, revert, restore, checkout; reflog — the safety net; Stash; Rewriting history; Secrets committed to history — the scenario question; git bisect — find the commit that broke it
- [[Git-Part-06-Tags-Submodules-Hooks-Worktrees-LFS|Git Part 6 — Tags, Submodules, Hooks, Worktrees, LFS]] — Tags and releases; Submodules; Hooks; Worktrees; Git LFS (Large File Storage)
- [[Git-Part-07-GitHub|Git Part 7 — GitHub]] — Repositories, forks and pull requests; Branch protection and repository rules; Issues, projects and templates; GitHub Actions (CI/CD); Other GitHub features
- [[Git-Part-08-Branching-Strategies|Git Part 8 — Branching Strategies]] — The three models; Branch naming conventions
- [[Git-Part-09-Git-in-DevOps|Git Part 9 — Git in DevOps]] — Git in CI/CD pipelines; GitOps; Git and Infrastructure as Code; Supply-chain security with Git
- [[Git-Part-10-Advanced-Internals-and-Performance|Git Part 10 — Advanced Internals & Performance]] — Packfiles and garbage collection; Large repository performance; Useful plumbing commands; Miscellaneous advanced features
- [[Git-Part-11-Interview-Question-Bank|Git Part 11 — Interview Question Bank]] — A. Basic (screening / fresher level); B. Intermediate; C. Advanced / senior; D. GitHub-specific; E. Scenario-based
- [[Git-Part-12-Cheat-Sheet|Git Part 12 — Cheat Sheet]] — Command → intent quick map
- [[Git-Part-13-What-Interviewers-Actually-Test|Git Part 13 — What Interviewers Actually Test]] — Phrases that read as senior; Red flags to avoid; Practice checklist

> [!note] Numbering
> The "Sections" above predate the final layout. In this vault: the interview question bank is [[Git-Part-11-Interview-Question-Bank|Part 11]], the cheat sheet is [[Git-Part-12-Cheat-Sheet|Part 12]], and there are 13 parts in total.

<!-- related -->
---
**Related:** [[3.1-Git-Workflow|Git Workflow for Real Teams]]
<!-- /related -->
