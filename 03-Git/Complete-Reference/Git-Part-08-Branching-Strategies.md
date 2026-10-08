---
title: Git Part 8 — Branching Strategies
type: reference
module: git
tags:
  - git
order: 8
up: "[[Git-GitHub-Complete-Reference-MOC]]"
prev: "[[Git-Part-07-GitHub]]"
next: "[[Git-Part-09-Git-in-DevOps]]"
related:
  - "[[3.1-Git-Workflow]]"
origin: "DevOps-Mastery/03-Git & GitHub/3.0-Git & GitHub — Complete Reference.md"
---

# Git Part 8 — Branching Strategies

<!-- nav -->
↑ [[Git-GitHub-Complete-Reference-MOC|Git & GitHub — Complete Reference]] · ← [[Git-Part-07-GitHub|Git Part 7 — GitHub]] · → [[Git-Part-09-Git-in-DevOps|Git Part 9 — Git in DevOps]]
<!-- /nav -->

## 38. The three models

### GitFlow

Branches: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`.

- `main` = production, tagged releases only
- `develop` = integration branch
- `release/*` = stabilisation before a release
- `hotfix/*` = urgent fix branched from `main`, merged into both `main` and `develop`

**Fits:** versioned/on-prem products, long QA cycles, multiple supported versions in the field. **Downsides:** heavy, slow, lots of long-lived branches, painful merges. Considered legacy for web/SaaS.

### GitHub Flow

`main` + short-lived feature branches. Everything merged to `main` is deployable and deployed. **Fits:** continuous deployment to a single environment. Simple and by far the most common.

### Trunk-Based Development

Everyone commits to `main` (trunk) or branches lasting under 24 hours. Incomplete work is hidden behind **feature flags**. Release branches are cut from trunk only when needed. **Fits:** high-velocity CI/CD, large teams, the 2025 default at scale (Google, Meta, Netflix). **Requires:** strong automated testing, feature flags, fast CI.

### GitLab Flow

GitHub Flow plus environment branches (`main` → `staging` → `production`) or release branches. A pragmatic middle ground.

### Comparison

||GitFlow|GitHub Flow|Trunk-Based|
|---|---|---|---|
|Long-lived branches|Many|One (`main`)|One (`main`)|
|Release cadence|Scheduled|Continuous|Continuous|
|Merge pain|High|Low|Lowest|
|Feature flags needed|No|Sometimes|Yes|
|Team size fit|Any|Small–medium|Medium–large|
|Complexity|High|Low|Low (process) / High (tooling)|

**Concrete recommendation:** for a containerised service deploying multiple times a day, use **trunk-based with feature flags**, short-lived branches, squash merges, and branch protection. Use GitFlow only if you genuinely ship versioned artifacts to customers and must support multiple releases simultaneously.

## 39. Branch naming conventions

```
feature/JIRA-123-add-login
bugfix/JIRA-456-fix-null-pointer
hotfix/critical-auth-bypass
release/v2.4.0
chore/upgrade-terraform-1.7
```

Keep it lowercase and hyphenated, with the ticket ID for traceability. Some teams enforce this with a `pre-push` hook or a CI check.

<!-- related -->
---
**Related:** [[3.1-Git-Workflow|Git Workflow for Real Teams]]
<!-- /related -->
