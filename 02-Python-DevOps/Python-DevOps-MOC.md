---
title: Python for DevOps — Map of Content
type: moc
module: python
tags:
  - moc
  - python
up: "[[Home]]"
---

# Python for DevOps — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

Python for automation: JSON/YAML, requests, subprocess, plus the Flask + MongoDB API that starts the golden-thread project.

**Related modules:** [[Linux-Bash-MOC|Linux & Bash]] · [[Docker-MOC|Docker]]

## Concepts
- [[2.1-JSON-YAML-Parsing|JSON & YAML Parsing]] — *concept* · interview: 🔴 High — YAML is everywhere in K8s, Terraform, CI/CD
  - Covers: JSON — Built-in, No Install Needed; YAML — Needs pyyaml; Navigating Nested Structures; Real DevOps Connection; Interview — Ready to Speak
- [[2.2-Python-Requests|Python — requests]] — *concept* · interview: 🟡 Medium — used for API interactions, webhooks, health checks
  - Covers: What Is It?; Basic Usage; With Headers & Auth; POST Request; Error Handling; Real DevOps Connection; Interview — Ready to Speak
- [[2.3-Python-Subprocess|Python — subprocess]] — *concept* · interview: 🔴 High — core tool for Python automation scripts
  - Covers: What Is It?; Basic Usage; With Error Handling; Real Example — Memory Usage; Common Gotcha; Interview — Ready to Speak

## Labs
- [[Lab-Python-DevOps-Scripts|Lab — Python DevOps Scripts]] — *lab*
  - Covers: Setup; Script 1 — Memory Usage via subprocess; Script 2 — YAML Manifest Parsing; Script 3 — API Request & JSON Navigation; Key Lessons from This Lab; boto3 — Deferred to AWS Week
- [[Lab-Flask-MongoDB-REST-API|Lab — Flask + MongoDB REST API]] — *lab* · interview: 🔴 High — REST API design, request lifecycle, DB operations
  - Covers: Architecture; How Flask Routing Works; Request → Flask → MongoDB Flow; Key Flask Objects; MongoDB Concepts; Environment Variable Config — 12-Factor App; Running the App

[[Home]] · [[Interview-Prep-MOC]]
