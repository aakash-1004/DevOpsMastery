---
title: Lab — Server Health Check Script
aliases:
  - Health-Check-Script
type: lab
module: linux
status: "✅ Completed & Working"
tags:
  - linux
  - bash
  - lab
  - day1
file: "`~/aakash/devops/health_check.sh`"
up: "[[Linux-Bash-MOC]]"
prev: "[[1.7-Log-Parsing]]"
next: "[[Lab-Postfix-mailx-Gmail-Relay]]"
related:
  - "[[1.5-Bash-Strict-Mode]]"
  - "[[Shell-08-Projects]]"
origin: DevOps-Mastery/01-Linux-Bash/Labs/Health-Check-Script.md
---

# Lab — Server Health Check Script

<!-- nav -->
↑ [[Linux-Bash-MOC|Linux & Bash]] · ← [[1.7-Log-Parsing|Log Parsing — grep, awk, sed]] · → [[Lab-Postfix-mailx-Gmail-Relay|Postfix + mailx Setup (Ubuntu, Gmail Relay)]]
<!-- /nav -->

## What It Does

- Checks CPU usage against a threshold (default: 80%)
- Checks memory usage against a threshold (default: 80%)
- Checks all disk partitions against a threshold (default: 90%)
- Checks if a target process (`dockerd`) is running
- Logs everything with timestamps to `/tmp/health_check.log`
- Uses lock file to prevent duplicate runs
- Cleans up lock file on any exit via `trap`

---

## Key Pattern — Run All Checks, Collect Failures

```bash
local FAILED=0
check_cpu     || FAILED=1
check_memory  || FAILED=1
check_disk    || FAILED=1
check_process || FAILED=1

if (( FAILED == 1 )); then
    log_error "Health check FAILED"
    exit 1
fi
```

**Why:** `set -e` would exit on the first failed check. `|| FAILED=1` overrides that — if the check fails, don't exit, just record it. All checks run, failures are collected, reported at the end.

---

## Bug Found During Lab

**Problem:** `PROCESS_NAME="docker"` → check always failed even though Docker was running.

**Root cause:** `pgrep -x` matches the exact process name. The Docker daemon process is `dockerd`, not `docker`.

**Fix:** `PROCESS_NAME="dockerd"`

**Lesson:** Always verify the real process name before hardcoding:
```bash
ps aux | grep docker
# or
pgrep -x dockerd
```

This type of bug is common in production — always verify assumptions.

---

## Test Cases Run

| Test | How | Expected Result |
|------|-----|----------------|
| Normal run | `./health_check.sh` | All checks pass |
| Process not running | `PROCESS_NAME="fakeprocess"` | Process check fails |
| Lock file active | `touch /tmp/health_check.lock` then run | Exits with "Previous run still active" |
| Low disk threshold | `DISK_THRESHOLD=1` | Disk check fails |

---

## Related notes
- [[1.5-Bash-Strict-Mode]]
- [[Lab-Logging-Trap-Lock-Files]]
- [[1.7-Log-Parsing]]]
- [[1.6-Cron-Jobs]]

<!-- related -->
---
**Related:** [[1.5-Bash-Strict-Mode|Bash Strict Mode]] · [[Shell-08-Projects|Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)]]
<!-- /related -->
