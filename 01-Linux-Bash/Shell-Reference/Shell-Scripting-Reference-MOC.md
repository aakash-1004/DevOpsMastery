---
title: Shell Scripting — Complete DevOps Reference
aliases:
  - 0.0- Shell Scripting-all
type: moc
module: linux
status: complete
tags:
  - devops
  - linux
  - bash
  - shell-scripting
  - interview-prep
  - moc
created: "2026-10-02"
level: mid-to-senior
up: "[[Linux-Bash-MOC]]"
related:
  - "[[1.2-Shell-Scripting]]"
  - "[[1.5-Bash-Strict-Mode]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Complete DevOps Reference

<!-- nav -->
↑ [[Linux-Bash-MOC|Linux & Bash]]
<!-- /nav -->

> Scope: everything a DevOps engineer with ~3 years of experience is expected to know, write, and defend in an interview. Format per topic: **Concept → Syntax → Gotchas → Interview Q&A** Everything marked **[MUST KNOW]** shows up in interviews. **[GOOD TO KNOW]** is depth, not table stakes.

## Contents

- [[Shell-01-Basics-Shells-Scripts-Shebang|Shell Scripting — Basics: Shells, First Script, Shebang, Running]] — How shell scripting is actually used in a DevOps job; What is a Linux Shell?; Types of Shell; How to check shell type; What is shell scripting?; Creating your first shell script
- [[Shell-02-Variables-Arrays-Strings-Input-Arithmetic|Shell Scripting — Variables, Arrays, Strings, Input, Arithmetic]] — Variables; Arrays; String operations; Getting input from the user; Arithmetic operations
- [[Shell-03-Conditionals-and-Logic|Shell Scripting — Conditionals, case, Logical Operators]] — If-else and test conditions; elif; case; Logical operators
- [[Shell-04-Loops|Shell Scripting — Loops]] — For loop; While, until, and infinite loops
- [[Shell-05-Functions-Arguments-and-Useful-Concepts|Shell Scripting — Functions, Arguments, Other Useful Concepts]] — Functions; Passing arguments to a script; Other useful concepts
- [[Shell-06-Real-Script-Special-Variables-Redirection|Shell Scripting — Real Script, Special Variables, Redirection, /dev/null]] — Real script — connectivity check; Special BASH variables; Redirection; /dev/null
- [[Shell-07-Logging-Debugging-and-Scheduling|Shell Scripting — Logging, Debugging, Scheduling (at, cron)]] — Logging in shell scripts; Debugging; Scheduling with `at`; Scheduling with crontab
- [[Shell-08-Projects|Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)]] — Project 1 — Monitor free RAM; Project 2 — Monitor disk space + email alert; Project 3 — Archive old logs; Project 4 — User management
- [[Shell-09-Production-Script-Template|Shell Scripting — Production Script Template]]
- [[Shell-10-Common-Gotchas-Cheat-Sheet|Shell Scripting — Common Gotchas Cheat Sheet]]
- [[Shell-11-Interview-Question-Bank|Shell Scripting — Interview Question Bank]] — Rapid-fire (one-liner answers expected); Conceptual; Scenario-based

## What To Drill Next

Ordered by interview return on investment:

1. **Write the production template from memory.** `set -euo pipefail`, trap cleanup, log function, SCRIPT_DIR, preflight checks. This is the single most testable thing.
2. **`while IFS= read -r line; do … done < file`** — say it in your sleep, and be able to explain each token.
3. **Redirection order** — `> file 2>&1` vs `2>&1 > file`, with the reasoning.
4. **The subshell/pipe gotcha** — why a counter in `cat f | while read` stays zero.
5. **Exit codes 127 / 137 / 143** and what they mean in Kubernetes.
6. **Cron failure checklist** — PATH, no profile, cwd, `/bin/sh`, `%` escaping, silent output.
7. **One real script you've written**, end to end, with a reason for each design choice. Interviewers push on this more than on syntax.

---

## Related notes

- [[1.4-Linux-Performance-Troubleshooting]]
- [[7.4-Jenkins-CICD-Pipeline]]
- [[Git-GitHub-Complete-Reference-MOC]]
- Docker-Entrypoint-and-Healthcheck-Patterns
- Kubernetes-Troubleshooting-CrashLoopBackOff
- Cron-vs-Systemd-Timers-vs-K8s-CronJob
- [[Interview-Prep-MOC]]

---

**Progress note:** Shell scripting — full topic coverage complete (fundamentals → projects → interview bank). Next: convert the 4 project scripts into a `aakash-1004/shell-scripting-labs` repo and run them on bumblebee for real output.

<!-- related -->
---
**Related:** [[1.2-Shell-Scripting|Shell Scripting]] · [[1.5-Bash-Strict-Mode|Bash Strict Mode]]
<!-- /related -->
