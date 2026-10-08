---
title: "Shell Scripting — Logging, Debugging, Scheduling (at, cron)"
type: reference
module: linux
status: complete
tags:
  - devops
  - linux
  - bash
  - shell-scripting
  - interview-prep
level: mid-to-senior
order: 7
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-06-Real-Script-Special-Variables-Redirection]]"
next: "[[Shell-08-Projects]]"
related:
  - "[[1.6-Cron-Jobs]]"
  - "[[Lab-Logging-Trap-Lock-Files]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Logging, Debugging, Scheduling (at, cron)

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-06-Real-Script-Special-Variables-Redirection|Shell Scripting — Real Script, Special Variables, Redirection, /dev/null]] · → [[Shell-08-Projects|Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)]]
<!-- /nav -->

## 27. Logging in shell scripts **[MUST KNOW]**

### Minimum viable logging function

```bash
LOG_FILE="/var/log/deploy.log"

log() {
  local level="$1"; shift
  printf '%s [%-5s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" | tee -a "$LOG_FILE" >&2
  # why: timestamp + level + message; tee writes to file AND stderr so it doesn't pollute captured stdout
}

log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@"; }
log_error() { log "ERROR" "$@"; }
die()       { log_error "$@"; exit 1; }

log_info "Deployment started by $(whoami) on $(hostname)"
log_error "Failed to pull image"
die "Cannot continue"
```

### Redirect the whole script's output to a log

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG="/var/log/backup-$(date +%F).log"
exec > >(tee -a "$LOG") 2>&1       # why: EVERYTHING from here on goes to the log AND the terminal
# why: >(...) is process substitution — tee runs as a co-process consuming the script's output

echo "This is logged automatically"
```

### Send to syslog / journald

```bash
logger -t backup-script "Backup completed"              # why: -t sets the tag so you can filter later
logger -p user.err -t backup-script "Backup FAILED"     # why: -p sets facility.priority

journalctl -t backup-script --since "1 hour ago"        # why: retrieve it
```

Using `logger` means your script's output lands in the same place as everything else on the box, so it's picked up by whatever ships logs to CloudWatch/ELK — no extra file to configure.

### Log rotation

A script writing to `/var/log/myapp.log` forever will eventually fill the disk. Add a logrotate config:

```
# /etc/logrotate.d/myscript
/var/log/myscript.log {
    daily
    rotate 7            # keep 7 days
    compress
    delaycompress
    missingok           # don't error if the file doesn't exist
    notifempty
    create 0640 root adm
}
```

### Structured logging for log aggregation **[GOOD TO KNOW]**

```bash
log_json() {
  printf '{"ts":"%s","level":"%s","script":"%s","msg":"%s"}\n' \
    "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$1" "${BASH_SOURCE[0]##*/}" "$2"
  # why: JSON lines parse cleanly in CloudWatch Insights / Loki / Elasticsearch without a custom grok pattern
}
log_json "ERROR" "deployment failed"
```

### Interview Q&A

**Q: How do you add logging to a shell script?**

> A `log()` function that prefixes an ISO timestamp and a level, writing to stderr and appending to a file via `tee`. Logs go to stderr specifically so they never contaminate stdout when the script's output is captured. For anything that runs unattended I also add a logrotate config, and if the logs feed an aggregator I emit JSON lines so they're parseable without a grok pattern.

---

## 28. Debugging **[MUST KNOW]**

```bash
bash -x script.sh      # why: trace — print each command after expansion, before execution
bash -v script.sh      # why: verbose — print each line as READ (before expansion)
bash -n script.sh      # why: syntax check only, DON'T execute — run this before deploying any script
bash -xv script.sh     # why: both
```

### Tracing a specific section only

```bash
set -x                 # why: enable tracing from here
risky_function
set +x                 # why: disable — ALWAYS turn it off before handling secrets
```

### Better trace output with PS4

```bash
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '
set -x
# Output becomes:  + deploy.sh:42:deploy_app: kubectl apply -f manifest.yaml
# why: default PS4 is just "+ " which tells you nothing about WHERE you are
```

### `trap` for debugging

```bash
trap 'echo "ERROR at line $LINENO: command was: $BASH_COMMAND" >&2' ERR
# why: $BASH_COMMAND holds the command that's about to run / just failed — pinpoints the failure exactly

trap 'read -rp "line $LINENO — press enter"' DEBUG   # why: poor-man's step debugger, fires before EVERY command
```

### ShellCheck — the single highest-value tool **[MUST KNOW]**

```bash
shellcheck script.sh                   # why: static analysis — catches unquoted vars, useless cat, bad [ ] usage
shellcheck -S error script.sh          # why: errors only
shellcheck -x script.sh                # why: follow `source`d files
# shellcheck disable=SC2086            # why: inline suppression when you deliberately want word splitting
```

**Put shellcheck in CI.** This is a strong, specific answer to "how do you ensure script quality?" — it catches the exact class of bug (unquoted variables, SC2086) that causes production incidents.

```yaml
# .github/workflows/lint.yml
- name: ShellCheck
  run: find . -name "*.sh" -print0 | xargs -0 shellcheck -S warning
```

### Debugging checklist for a script that "works locally but fails in cron/CI"

1. `PATH` — cron has a minimal PATH. Use absolute paths or set PATH at the top.
2. Environment variables — cron doesn't source `.bashrc` or `.bash_profile`.
3. Working directory — cron starts in `$HOME`, not the script's directory. Use the `SCRIPT_DIR` idiom.
4. No TTY — anything interactive (`read`, `sudo` password, colour output) breaks.
5. Shell — cron uses `/bin/sh` unless you set `SHELL=/bin/bash` in the crontab.
6. User — the cron user may lack permissions, or have different credentials (no `~/.aws/credentials`).
7. Locale — `LANG` differs, which changes `date` and `sort` output.

Being able to rattle this list off is a strong signal in an interview.

### Interview Q&A

**Q: How do you debug a shell script?**

> `bash -n` first for a syntax check without running it. Then `bash -x`, usually with a custom `PS4` so the trace shows file, line, and function instead of a bare `+`. For intermittent failures I add `trap 'echo "failed at $LINENO: $BASH_COMMAND"' ERR`, which names the exact failing command. And I run shellcheck — most bugs I'd otherwise hunt manually are things it flags statically.

**Q: Script works when you run it, fails under cron. Walk me through it.**

> Cron gives you a minimal environment: a short PATH, no `.bashrc`, working directory is `$HOME`, and `/bin/sh` as the shell. My first move is to capture the real failure — change the crontab entry to `>> /tmp/debug.log 2>&1` instead of discarding output. Then I check PATH and absolute paths, confirm the script resolves its own directory rather than assuming cwd, and verify the cron user actually has the credentials and permissions the script needs. Nine times out of ten it's PATH or a missing environment variable.

---

## 29. Scheduling with `at` **[GOOD TO KNOW]**

One-off scheduled execution, as opposed to cron's recurring schedule.

```bash
at 14:30                        # why: interactive — type commands, Ctrl+D to submit
at now + 1 hour
at 10:00 AM tomorrow
at midnight
at 09:00 2026-10-15

echo "/opt/scripts/restart.sh" | at 02:00                 # why: non-interactive, pipe the command in
at -f /opt/scripts/maintenance.sh 03:00                   # why: -f run a script file

atq                             # why: list pending jobs
atrm 3                          # why: remove job number 3
at -c 3                         # why: show the full content of job 3, including the captured environment
```

`at` captures your **current environment** at submission time, which cron does not. Requires `atd` running: `systemctl enable --now atd`.

**Real use case:** scheduling a one-time maintenance restart during a change window, or delaying a risky command so you can cancel it. Classic sysadmin trick: before changing firewall rules remotely, `echo "iptables -F" | at now + 5 minutes` so you don't lock yourself out.

---

## 30. Scheduling with crontab **[MUST KNOW]**

```
┌─ minute (0-59)
│ ┌─ hour (0-23)
│ │ ┌─ day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command
```

```bash
crontab -e            # why: edit YOUR crontab (validates syntax on save — never edit the spool file directly)
crontab -l            # why: list
crontab -r            # why: DELETE ALL — dangerous, no confirmation. Back up with `crontab -l > backup` first.
crontab -u deploy -l  # why: another user's crontab (needs root)
sudo vim /etc/crontab # why: SYSTEM crontab — has an extra USER field before the command
```

### Common schedules

```bash
*/5 * * * *     # every 5 minutes
0 * * * *       # every hour, on the hour
0 2 * * *       # daily at 02:00
0 2 * * 0       # weekly, Sunday 02:00
0 2 1 * *       # monthly, 1st at 02:00
0 9-17 * * 1-5  # hourly, 9am-5pm, Mon-Fri
@reboot         # once at boot
@daily @weekly @monthly @hourly   # shorthand
```

### Production crontab — what it should actually look like

```bash
SHELL=/bin/bash                                   # why: cron defaults to /bin/sh (dash) — bash syntax would break
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=platform-alerts@example.com                # why: cron emails stdout/stderr here; empty MAILTO= disables mail

0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
# why: ABSOLUTE path to script, ABSOLUTE path to log, capture BOTH streams

*/5 * * * * /usr/bin/flock -n /var/lock/sync.lock /opt/scripts/sync.sh >> /var/log/sync.log 2>&1
# why: flock prevents overlapping runs if sync.sh takes longer than 5 minutes

0 3 * * * /opt/scripts/report.sh 2>&1 | logger -t report-script
# why: route output to syslog instead of a file so your log shipper picks it up
```

### Cron gotchas **[these are the interview answers]**

1. **PATH is minimal** — typically `/usr/bin:/bin`. `docker`, `kubectl`, `aws` won't be found. Use absolute paths or set PATH in the crontab.
2. **No shell profile** — `.bashrc` / `.bash_profile` are never sourced. Environment variables you rely on don't exist.
3. **`%` is special** — it's interpreted as a newline. `date +%F` **must** be escaped as `date +\%F` in a crontab line. This bites everyone once.
4. **Working directory is `$HOME`** — relative paths break.
5. **Default shell is `/bin/sh`** — set `SHELL=/bin/bash` if you use bash syntax.
6. **Silent failure** — without redirection, output goes to local mail that nobody reads. Always redirect to a log.
7. **No overlap protection** — a slow job will stack. Use `flock`.
8. **Timezone** — cron uses the system timezone. Check with `timedatectl`. A UTC server runs your "2 AM" job at 7:30 AM IST.

### systemd timers — the modern alternative **[GOOD TO KNOW, increasingly asked]**

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
User=backup
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true            # why: run on next boot if the machine was off at the scheduled time — cron CANNOT do this
RandomizedDelaySec=300     # why: jitter so 500 servers don't hit the backup target simultaneously

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now backup.timer
systemctl list-timers              # why: see next/last run for all timers
journalctl -u backup.service       # why: logs are captured automatically — no redirection needed
```

**Why systemd timers over cron:** automatic logging to journald, dependency ordering, resource limits, `Persistent=true` for missed runs, and randomized delay. Cron wins on simplicity and universality.

### Interview Q&A

**Q: Your cron job doesn't run. How do you debug it?**

> First confirm cron even fired it — `grep CRON /var/log/syslog` or `journalctl -u cron`. If it fired, the script failed. I change the entry to redirect both streams to a file, run it, and read the actual error. The usual suspects are PATH (binaries not found), the working directory being `$HOME` instead of the script's directory, missing environment variables because profiles aren't sourced, and an unescaped `%` in a `date` format string.

**Q: How do you stop a cron job running twice if it overruns its schedule?**

> Wrap it in `flock -n /var/lock/job.lock`. Non-blocking, so the second instance exits immediately rather than queueing. Without it, a 7-minute job on a 5-minute schedule accumulates processes until the box falls over.

**Q: Cron vs systemd timers?**

> Cron is simpler and available everywhere, so it's still fine for basic housekeeping. systemd timers give you journald logging for free, dependency ordering, `Persistent=true` to catch runs missed while the machine was down, and randomized delays to avoid thundering herds across a fleet. On modern Linux for anything that matters, I'd use a timer. In Kubernetes neither applies — that's a CronJob.

<!-- related -->
---
**Related:** [[1.6-Cron-Jobs|Cron Jobs]] · [[Lab-Logging-Trap-Lock-Files|Logging, Trap & Lock Files]]
<!-- /related -->
