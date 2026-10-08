---
title: Vault Guide
type: reference
module: home
tags:
  - guide
  - meta
---

# Vault Guide

This vault is the cleaned-up version of the `DevOps` repo notes. Start at [[Home]].

## Folder map

| Folder | Module hub |
|---|---|
| `00-Home` | [[Home]], [[Progress-Tracker]], [[Golden-Thread-Project]], [[Quick-Commands]], [[Cross-Cloud-Service-Map]], this guide, [[Migration-Map]], [[Backlog-Planned-Notes]] |
| `00-Fundamentals` | [[Fundamentals-MOC]] |
| `01-Linux-Bash` | [[Linux-Bash-MOC]] |
| `02-Python-DevOps` | [[Python-DevOps-MOC]] |
| `03-Git` | [[Git-MOC]] |
| `04-Docker` | [[Docker-MOC]] |
| `05-Kubernetes` | [[Kubernetes-MOC]] |
| `06-AWS` | [[AWS-MOC]] |
| `07-CICD` | [[CICD-MOC]] |
| `08-Terraform` | [[Terraform-MOC]] |
| `09-Monitoring` | [[Monitoring-MOC]] |
| `10-Deployment` | [[Deployment-MOC]] |
| `11-Azure` | [[Azure-MOC]] |
| `12-GCP` | [[GCP-MOC]] |
| `13-Interview-Prep` | [[Interview-Prep-MOC]] |
| `14-Revision` | [[Master-Revision-MOC]] |
| `99-Templates` | note templates (Templates core plugin points here) |

## Conventions

- **Hubs (MOCs):** every module has a `*-MOC` note listing its notes (grouped, with what each covers). Big references (AWS, Azure, GCP, Git, Shell, Revision, Boardgame project, Terraform course) have their own hub with a numbered contents list.
- **Navigation:** every note carries `up`, `prev`, `next` properties and a one-line breadcrumb under its title (Terraform course notes keep their original "Back to / Next" lines). `related` holds cross-module links.
- **Naming:** curriculum notes keep their original numbers (`4.2-…`, `5.1-…`); labs/projects live in each module's `Labs/` folder as `Lab-…` / `Project-…`; parts of split references are `AWS-03-IAM`, `Git-Part-05-…`, `Shell-04-Loops`, etc. Old file names are kept as `aliases`, so searching or linking the old name still finds the note.
- **Properties:** `title`, `aliases`, `type` (concept · reference · lab · project · interview · cheatsheet · moc · hub), `module`, `status`, `interview-relevance`, `tags`, optional `date`/`created`/`repo`/`docker-hub`/`instance`/`cluster`/`file(s)`/`config`, plus `origin` = the path in the old repo.
- **Metadata moved, not lost:** the old `**Tags:** … **Status:** …` lines and `## Meta` blocks are now YAML properties.
- **Generated blocks:** the breadcrumb and the "Related" footer are wrapped in `<!-- nav -->` / `<!-- related -->` comments so they can be regenerated without touching your text.
- **Planned notes:** links to notes that never existed are no longer broken links; they are listed in [[Backlog-Planned-Notes]].

## What the cleanup changed (nothing else was edited)

- **223 notes** in the vault (from 98 markdown files in the repo): 110 are parts split out of the big references.
- **Metadata → properties** on every note (tags, status, interview relevance, repo, dates, …). The inline `**Tags:** …` lines, `## Meta` bullet lists and `## tags: …` one-liners are gone from the bodies.
- **Broken front-matter fixed** in 5 notes (a bare `---` / `## tags: [...]` block that Obsidian would have swallowed or shown as text): `0.0 How to write.md`, `7.4.1-Jenkins — Controller&Agent Architecture.md`, `7.4.2- Jenkins block2 jenkinsfile anatomy.md`, `GoWebApp-Full-GitOps-Pipeline.md.md`, `Boardgame Jenkins CICD Pipeline.md`.
- **Escaped code fences** (`\`\`\``) in `5.5-EKS-Fargate-IRSA-ALB-Project` restored to real fences — the whole command block was rendering as plain text with headings.
- **Tab-indented titles** (rendered as code blocks) fixed in: `1.4-Linux-Performance-Troubleshooting.md.md`, `4.3-Docker-Images-Containers(Deep).md`, `7.4-Jenkins-CICD-Notes.md`.
- **10 files named `*.md.md`** renamed; `(Deep)` / spaces / `&` removed from file names (old names kept as aliases).
- **Dead chat-export links removed:** 31 `claude.ai/chat/…` links in the AWS and Master-Revision tables of contents were replaced by wikilinked contents lists in their hubs. Azure/GCP in-page `(#anchor)` TOCs likewise.
- **Wikilinks:** all 1946 links in the vault resolve. Links with `.md`, folder prefixes or old names were re-pointed; 18 targets that never existed are listed in [[Backlog-Planned-Notes]] instead of staying broken.
- **Title cleanups:** `07 — GitHub Actions CI/CD Pipeline`, `07 — CI/CD: ArgoCD & GitOps` (both numbered 07) and `CI/CD: Helm` (which is a Kubernetes note) got plain titles; the old titles are aliases.
- **`7.3-ArgoCD-and-GitOps` split in three:** concepts stay in 7.3; the Helm part is [[7.3.1-Helm-Chart-Anatomy-and-Commands]]; the ArgoCD part (manifest, commands, spoken answers) is [[7.3.2-ArgoCD-Application-Manifest-and-Commands]].
- **Progress-Tracker:** System Design → 📝 Notes written, Interview Prep → 🔄 Ongoing, "System Design" next-step ticked. Confidence table left as you wrote it.
- `Logging-Trap-Lock-Files` moved from 02-Python-DevOps to 01-Linux-Bash/Labs (it is bash content; the Dashboard also listed it under Linux).
- **Dashboard** split into [[Home]], [[Progress-Tracker]], [[Golden-Thread-Project]], [[Quick-Commands]]; its hand-written "Notes Index" is replaced by the hubs.
- Runs of 3+ blank lines collapsed; no other body text was changed. `Migration-Map` + `origin` properties give the audit trail.

## Things in the original notes that I left untouched but you may want to fix

- **Git reference preamble** says "Sections 1–4 … Sections 12–16 … Section 17 = interview question bank, Section 18 = cheat sheet", but the reference has 13 parts (interview bank = Part 11, cheat sheet = Part 12). Original text kept in [[Git-GitHub-Complete-Reference-MOC]]; the mapping is noted there.
- **10-Deployment** jumps from `10.4` to `10.6` (no 10.5 existed in the repo); numbering kept as it was.
- Several notes dated 2026-05 (Helm, ArgoCD, GitHub Actions pipeline) use the old "Meta" layout while newer ones use YAML — now unified, but the `interview-relevance` values still mix emoji (🔴 High), stars (⭐⭐⭐⭐⭐, ★★★★★) and `5/5` because I kept your wording.

## Notes DB

The vault ships with `Notes-Database.base` (Obsidian core "Bases" plugin) — a live table of every note by module, type, status and interview relevance, replacing the hand-maintained table that used to live in the Dashboard. I could not open Obsidian in this environment, so that file is the one thing in this vault I have not been able to render-test.
