---
title: "Logging, Trap & Lock Files"
aliases:
  - Logging-Trap-Lock-Files
type: lab
module: linux
status: ✅ Understood
interview-relevance: "🔴 High — production scripting pattern, shows maturity"
tags:
  - linux
  - bash
  - scripting
  - day1
  - lab
up: "[[Linux-Bash-MOC]]"
prev: "[[Lab-Postfix-mailx-Gmail-Relay]]"
related:
  - "[[1.5-Bash-Strict-Mode]]"
  - "[[Shell-07-Logging-Debugging-and-Scheduling]]"
origin: DevOps-Mastery/02-Python-DevOps/Labs/Logging-Trap-Lock-Files.md
---

# Logging, Trap & Lock Files

<!-- nav -->
↑ [[Linux-Bash-MOC|Linux & Bash]] · ← [[Lab-Postfix-mailx-Gmail-Relay|Postfix + mailx Setup (Ubuntu, Gmail Relay)]]
<!-- /nav -->

## Logging Functions

```bash
LOG_FILE="/tmp/health_check.log"

log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}
log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}
```

**Key concepts:**
- `tee -a` → prints to terminal AND appends to log file simultaneously
- `$()` → command substitution — runs `date` and inserts output inline
- `$1` → first argument passed to the function
- Production standard: errors should go to `stderr` with `>&2`

---

## Trap & Cleanup

```bash
cleanup() {
    rm -f "$LOCKFILE"
}
trap cleanup EXIT
```

**How it works:**
- `trap <function> <signal>` — runs function when signal fires
- `EXIT` fires on ANY exit — normal, error, or Ctrl+C
- Guarantees cleanup always happens, even on crash

**Common trap signals:**

| Signal | When It Fires |
|--------|--------------|
| `EXIT` | Any exit — normal or error |
| `INT` | Ctrl+C |
| `TERM` | `kill` command |
| `ERR` | Any command error (with `-e`) |

---

## Lock File Pattern

```bash
LOCKFILE="/tmp/health_check.lock"

if [[ -f "$LOCKFILE" ]]; then
    log_error "Previous run still active. Exiting."
    exit 1
fi
touch "$LOCKFILE"
```

**Why:** Prevents two instances of the same script running simultaneously. Critical for cron jobs.

**Flow:**
1. Script starts → check if lock file exists
2. If yes → another instance is running → exit immediately
3. If no → create lock file with `touch`, proceed
4. On any exit → `trap cleanup EXIT` removes the lock file

**Gotcha:** Always use `-f` not `-e`. `-f` checks file exists AND is a regular file.

---

## Interview — Ready to Speak

**Q: "How do you make sure a cron script doesn't run twice simultaneously?"**

> "Lock file pattern. At the start of the script, check if a lock file exists — if it does, another instance is running, so exit. If not, create the lock file and proceed. The key is pairing this with `trap cleanup EXIT` so the lock file is always deleted on exit, even if the script crashes. Without the trap, a crashed script would leave the lock file behind and block all future runs."

---

## Related notes
- [[1.5-Bash-Strict-Mode|Bash-Strict-Mode.md]]
- [[1.7-Log-Parsing]]
- [[1.6-Cron-Jobs]]

<!-- related -->
---
**Related:** [[1.5-Bash-Strict-Mode|Bash Strict Mode]] · [[Shell-07-Logging-Debugging-and-Scheduling|Shell Scripting — Logging, Debugging, Scheduling (at, cron)]]
<!-- /related -->
