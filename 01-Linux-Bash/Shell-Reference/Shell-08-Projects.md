---
title: "Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)"
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
order: 8
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-07-Logging-Debugging-and-Scheduling]]"
next: "[[Shell-09-Production-Script-Template]]"
related:
  - "[[Lab-Server-Health-Check-Script]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-07-Logging-Debugging-and-Scheduling|Shell Scripting — Logging, Debugging, Scheduling (at, cron)]] · → [[Shell-09-Production-Script-Template|Shell Scripting — Production Script Template]]
<!-- /nav -->

These four are the standard "prove you can write bash" exercises. Each one below is the production-grade version, not the tutorial version — that's the difference an interviewer notices.

---

## Project 1 — Monitor free RAM

```bash
#!/usr/bin/env bash
#
# check-memory.sh — alert when available memory drops below a threshold
# Usage : ./check-memory.sh [threshold_percent]
# Cron  : */5 * * * * /opt/scripts/check-memory.sh 15 >> /var/log/memcheck.log 2>&1
# Exit  : 0 ok | 1 below threshold
#
set -euo pipefail

THRESHOLD="${1:-20}"                  # why: default 20% available; overridable per environment
HOSTNAME_S="$(hostname -s)"
LOG_TAG="mem-check"

# why: `available` (column 7) is the number that matters, NOT `free`.
# why: Linux uses spare RAM for page cache; `free` looks alarmingly low on a healthy box.
read -r TOTAL AVAIL <<< "$(free -m | awk 'NR==2 {print $2, $7}')"
# why: <<< here-string + read assigns two awk outputs into two variables in one go

AVAIL_PCT=$(awk -v a="$AVAIL" -v t="$TOTAL" 'BEGIN { printf "%.1f", (a/t)*100 }')
# why: awk because bash can't do floating point

logger -t "$LOG_TAG" "available=${AVAIL}MB total=${TOTAL}MB pct=${AVAIL_PCT}%"

# why: awk for the comparison too, since AVAIL_PCT is a decimal and (( )) can't handle it
if awk -v p="$AVAIL_PCT" -v t="$THRESHOLD" 'BEGIN { exit !(p < t) }'; then
  echo "[CRITICAL] $HOSTNAME_S memory available ${AVAIL_PCT}% (${AVAIL}MB/${TOTAL}MB) < ${THRESHOLD}%"

  echo "--- Top 5 memory consumers ---"
  ps -eo pid,ppid,user,%mem,rss,comm --sort=-%mem | head -6
  # why: include diagnostic context in the alert — an alert without context causes a second investigation

  echo "--- Swap ---"
  free -m | awk 'NR==3 {print "used:", $3"MB", "free:", $4"MB"}'

  exit 1
fi

echo "[OK] $HOSTNAME_S memory available ${AVAIL_PCT}% (${AVAIL}MB/${TOTAL}MB)"
exit 0
```

**The point to make in an interview:** monitoring `free` instead of `available` produces constant false alarms, because Linux deliberately fills unused RAM with page cache and reclaims it on demand. Always use `available` (free -m column 7, or `MemAvailable` in `/proc/meminfo`).

---

## Project 2 — Monitor disk space + email alert

```bash
#!/usr/bin/env bash
#
# check-disk.sh — alert on filesystems above a usage threshold
# Usage : ./check-disk.sh [threshold_percent] [email]
# Cron  : 0 * * * * /opt/scripts/check-disk.sh 85 ops@example.com >> /var/log/diskcheck.log 2>&1
#
set -euo pipefail

THRESHOLD="${1:-85}"
ALERT_EMAIL="${2:-${ALERT_EMAIL:-}}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"           # why: optional — modern teams alert to Slack, not email
HOST="$(hostname -f)"
ALERTS=()                                    # why: collect all breaches, send ONE alert, not one per filesystem

# why: -P forces POSIX single-line output (long device names otherwise wrap onto 2 lines and break awk)
# why: -x tmpfs -x devtmpfs excludes pseudo-filesystems that are always "full" and irrelevant
while read -r fs size used avail pct mount; do
  pct="${pct%\%}"                            # why: strip the trailing % so it's a number
  if (( pct >= THRESHOLD )); then
    ALERTS+=("$(printf '%-28s %-10s %5s%% used (%s free)' "$mount" "$fs" "$pct" "$avail")")
  fi
done < <(df -PT -x tmpfs -x devtmpfs -x overlay | awk 'NR>1 {print $1, $3, $4, $5, $6, $7}')
# why: process substitution, NOT a pipe — a pipe would run the loop in a subshell and ALERTS would be empty

# Also check inodes — a filesystem can be 30% full and still fail with "No space left on device"
while read -r fs ipct mount; do
  ipct="${ipct%\%}"
  [[ "$ipct" == "-" ]] && continue           # why: some filesystems report no inode stats
  if (( ipct >= THRESHOLD )); then
    ALERTS+=("$(printf '%-28s %-10s %5s%% INODES used' "$mount" "$fs" "$ipct")")
  fi
done < <(df -PTi -x tmpfs -x devtmpfs | awk 'NR>1 {print $1, $6, $7}')

if (( ${#ALERTS[@]} == 0 )); then
  echo "[OK] all filesystems below ${THRESHOLD}%"
  exit 0
fi

BODY="Disk usage alert on ${HOST} at $(date '+%F %T %Z')
Threshold: ${THRESHOLD}%

$(printf '%s\n' "${ALERTS[@]}")

Largest directories under the affected mounts:
$(du -xh --max-depth=2 / 2>/dev/null | sort -rh | head -10)"
# why: -x stays on one filesystem; sort -rh sorts human-readable sizes descending

echo "$BODY"

if [[ -n "$ALERT_EMAIL" ]]; then
  if command -v mail >/dev/null 2>&1; then
    echo "$BODY" | mail -s "[DISK ALERT] ${HOST} >= ${THRESHOLD}%" "$ALERT_EMAIL"
    # why: requires mailutils/postfix configured. On EC2, outbound port 25 is blocked by default —
    # why: use SES, or skip email entirely and use a webhook.
  else
    echo "WARN: mail command not available, skipping email" >&2
  fi
fi

if [[ -n "$SLACK_WEBHOOK" ]]; then
  payload=$(printf '{"text":"*DISK ALERT* %s\n```%s```"}' "$HOST" "$(printf '%s\n' "${ALERTS[@]}")")
  curl -sf -X POST -H 'Content-type: application/json' --data "$payload" "$SLACK_WEBHOOK" >/dev/null \
    || echo "WARN: slack notification failed" >&2
  # why: || guard so a webhook outage doesn't make the whole check script fail
fi

exit 1
```

**Two things to mention in an interview:**

1. **Check inodes, not just blocks.** A box with millions of small session files can hit 100% inodes at 20% disk usage and throw "No space left on device" — a confusing incident if you only ever look at `df -h`.
2. **Email from EC2 doesn't just work.** AWS blocks outbound port 25 by default. The realistic answer in 2026 is a Slack/Teams webhook or SNS, not `mail`.

---

## Project 3 — Archive old logs

```bash
#!/usr/bin/env bash
#
# archive-logs.sh — compress logs older than N days, delete archives older than M days
# Usage : ./archive-logs.sh [log_dir] [compress_after_days] [delete_after_days]
# Cron  : 0 1 * * * /usr/bin/flock -n /var/lock/archlogs.lock /opt/scripts/archive-logs.sh
#
set -euo pipefail

LOG_DIR="${1:-/var/log/myapp}"
COMPRESS_DAYS="${2:-7}"
DELETE_DAYS="${3:-90}"
ARCHIVE_DIR="${LOG_DIR}/archive"
S3_BUCKET="${S3_BUCKET:-}"                   # why: optional offload to S3 before local deletion
DRY_RUN="${DRY_RUN:-false}"

log() { printf '%s [%s] %s\n' "$(date '+%F %T')" "$1" "${*:2}"; }

run() {
  if [[ "$DRY_RUN" == "true" ]]; then
    log INFO "DRY-RUN: $*"                   # why: a destructive script MUST have a dry-run mode
  else
    "$@"
  fi
}

[[ -d "$LOG_DIR" ]] || { log ERROR "Directory not found: $LOG_DIR"; exit 1; }
mkdir -p "$ARCHIVE_DIR"                      # why: -p is idempotent

COMPRESSED=0
SKIPPED=0

# why: -print0 / IFS= read -d '' handles filenames with spaces or newlines safely
# why: -type f excludes directories; ! -name "*.gz" avoids re-compressing
# why: -mtime +N means "modified more than N*24h ago"
while IFS= read -r -d '' file; do
  base="${file##*/}"                         # why: pure-bash basename, no subprocess

  # Skip files still held open by a running process — compressing these loses data
  if lsof -- "$file" >/dev/null 2>&1; then   # why: lsof shows open file handles
    log WARN "In use, skipping: $base"
    (( SKIPPED++ ))
    continue
  fi

  dest="${ARCHIVE_DIR}/${base}.$(date -r "$file" +%Y%m%d).gz"
  # why: date -r uses the FILE's mtime, not today's date — so the archive name reflects the log's actual day

  if run gzip -c "$file" > "$dest" 2>/dev/null && run rm -f "$file"; then
    # why: gzip -c to a new path then remove, rather than gzip in place — safer if the fs fills mid-write
    log INFO "Archived: $base -> ${dest##*/}"
    (( COMPRESSED++ ))
  else
    log ERROR "Failed to archive: $base"
    rm -f "$dest"                            # why: clean up a partial archive
  fi
done < <(find "$LOG_DIR" -maxdepth 1 -type f -name "*.log" ! -name "*.gz" -mtime "+${COMPRESS_DAYS}" -print0)

# Offload to S3 before local deletion
if [[ -n "$S3_BUCKET" ]]; then
  run aws s3 sync "$ARCHIVE_DIR" "s3://${S3_BUCKET}/$(hostname -s)/logs/" --only-show-errors
  # why: sync is idempotent — re-running uploads only what changed
fi

# Purge archives past retention
DELETED=$(find "$ARCHIVE_DIR" -type f -name "*.gz" -mtime "+${DELETE_DAYS}" -print | wc -l)
if [[ "$DRY_RUN" != "true" ]]; then
  find "$ARCHIVE_DIR" -type f -name "*.gz" -mtime "+${DELETE_DAYS}" -delete
  # why: -delete is safer than | xargs rm — no filename-parsing issues at all
fi

log INFO "Summary: compressed=$COMPRESSED skipped=$SKIPPED deleted=$DELETED"
```

**Interview points:**

- **`-print0` + `read -d ''`** is the only correct way to iterate find results. Say this and you've answered a filename-with-spaces question before it's asked.
- **`lsof` check** — gzipping a log a process still holds open means the process keeps writing to a deleted inode, and you lose the data. This is a real production incident.
- **Dry-run mode is mandatory** for anything that deletes.
- **The real answer is logrotate.** Mention that you'd normally use logrotate for this and only hand-roll a script when the requirement is unusual (S3 offload, per-tenant retention). Knowing when _not_ to write a script is a senior signal.

---

## Project 4 — User management

```bash
#!/usr/bin/env bash
#
# manage-users.sh — bulk create/delete/audit users from a CSV
# Usage : ./manage-users.sh create users.csv
#         ./manage-users.sh delete users.csv
#         ./manage-users.sh audit
# CSV   : username,fullname,groups(semicolon-separated),shell
#
set -euo pipefail

ACTION="${1:?Usage: $0 {create|delete|audit} [csvfile]}"
CSV="${2:-}"
LOG="/var/log/user-management.log"

log() { printf '%s [%s] %s\n' "$(date '+%F %T')" "$1" "${*:2}" | tee -a "$LOG"; }

(( EUID == 0 )) || { log ERROR "Must run as root"; exit 1; }
# why: EUID is the EFFECTIVE uid — the correct root check; comparing $USER to "root" fails under sudo -u

user_exists() { id "$1" &>/dev/null; }      # why: id returns non-zero if the user doesn't exist — cleaner than grepping /etc/passwd

create_users() {
  [[ -f "$CSV" ]] || { log ERROR "CSV not found: $CSV"; exit 1; }

  while IFS=, read -r username fullname groups shell || [[ -n "$username" ]]; do
    [[ -z "$username" || "$username" =~ ^[[:space:]]*# ]] && continue   # why: skip blanks and comments

    # Validate the username against the POSIX portable format before touching the system
    if ! [[ "$username" =~ ^[a-z_][a-z0-9_-]{0,31}$ ]]; then
      log ERROR "Invalid username format: $username"
      continue
    fi

    if user_exists "$username"; then
      log WARN "Exists, skipping: $username"          # why: IDEMPOTENT — re-running is safe
    else
      useradd -m -c "$fullname" -s "${shell:-/bin/bash}" "$username"
      # why: -m creates the home dir, -c sets the GECOS comment, -s sets the login shell
      log INFO "Created user: $username"

      # Force a password change on first login instead of setting a password here
      passwd -l "$username"                            # why: LOCK the password — SSH key auth only
      chage -d 0 "$username" 2>/dev/null || true       # why: expire immediately so any password set later must be changed
      # why: NEVER generate and log a plaintext password — it ends up in your log aggregator forever
    fi

    # Groups — idempotent, runs whether the user is new or existing
    if [[ -n "$groups" ]]; then
      IFS=';' read -ra GROUP_LIST <<< "$groups"
      for g in "${GROUP_LIST[@]}"; do
        getent group "$g" >/dev/null || { log WARN "Group missing, creating: $g"; groupadd "$g"; }
        usermod -aG "$g" "$username"       # why: -aG APPENDS. Without -a, usermod -G REPLACES all secondary groups.
        log INFO "Added $username to $g"
      done
    fi

    # SSH key setup — the correct way to grant access
    local home; home=$(getent passwd "$username" | cut -d: -f6)
    install -d -m 700 -o "$username" -g "$username" "${home}/.ssh"
    # why: install -d creates the dir with exact mode and ownership in one atomic step
    if [[ -f "keys/${username}.pub" ]]; then
      install -m 600 -o "$username" -g "$username" "keys/${username}.pub" "${home}/.ssh/authorized_keys"
      log INFO "Installed SSH key for $username"       # why: sshd REFUSES keys if perms are wrong: 700 dir, 600 file
    fi

  done < "$CSV"
}

delete_users() {
  [[ -f "$CSV" ]] || { log ERROR "CSV not found: $CSV"; exit 1; }

  while IFS=, read -r username _ || [[ -n "$username" ]]; do
    [[ -z "$username" || "$username" =~ ^[[:space:]]*# ]] && continue
    user_exists "$username" || { log WARN "Not found: $username"; continue; }

    # Safety rails — never delete system accounts
    local uid; uid=$(id -u "$username")
    if (( uid < 1000 )); then
      log ERROR "Refusing to delete system account (UID $uid): $username"
      continue                                         # why: UID < 1000 = system account on most distros
    fi

    # Archive the home directory before deletion — deleting data with no backup is unrecoverable
    if [[ -d "/home/$username" ]]; then
      tar -czf "/var/backups/${username}-$(date +%F).tar.gz" -C /home "$username" 2>/dev/null \
        && log INFO "Archived home: $username"
    fi

    pkill -u "$username" 2>/dev/null || true           # why: userdel fails if the user has running processes
    sleep 1
    pkill -9 -u "$username" 2>/dev/null || true

    userdel -r "$username" && log INFO "Deleted: $username"   # why: -r removes home dir and mail spool
  done < "$CSV"
}

audit_users() {
  echo "=== Interactive accounts (UID >= 1000) ==="
  awk -F: '$3 >= 1000 && $3 < 65534 {printf "%-20s uid=%-6s shell=%s\n", $1, $3, $7}' /etc/passwd

  echo
  echo "=== Accounts with EMPTY password (critical) ==="
  awk -F: '$2 == "" {print $1}' /etc/shadow            # why: empty password field = passwordless login

  echo
  echo "=== UID 0 accounts (should ONLY be root) ==="
  awk -F: '$3 == 0 {print $1}' /etc/passwd             # why: a second UID-0 account is a classic backdoor

  echo
  echo "=== Members of sudo/wheel ==="
  getent group sudo wheel 2>/dev/null | cut -d: -f1,4

  echo
  echo "=== Never logged in ==="
  lastlog -b 90 2>/dev/null | grep -v "Never logged in" | tail -n +2 || true

  echo
  echo "=== Accounts with authorized_keys ==="
  for h in /home/*/; do
    u="${h%/}"; u="${u##*/}"
    [[ -f "${h}.ssh/authorized_keys" ]] && \
      echo "$u: $(grep -c . "${h}.ssh/authorized_keys" 2>/dev/null || echo 0) key(s)"
  done
}

case "$ACTION" in
  create) create_users ;;
  delete) delete_users ;;
  audit)  audit_users ;;
  *) echo "Usage: $0 {create|delete|audit} [csvfile]" >&2; exit 1 ;;
esac
```

**Interview points:**

- **`usermod -aG` vs `usermod -G`** — forgetting `-a` wipes every other secondary group. People have locked themselves out of sudo this way. This is a commonly asked gotcha.
- **Never generate or log plaintext passwords.** Lock the password, use SSH keys, and let the identity provider handle auth.
- **SSH key permissions:** `.ssh` must be 700, `authorized_keys` 600, owned by the user. sshd silently refuses the key otherwise — a very common "key auth doesn't work" root cause.
- **UID < 1000 guard** — a bulk delete script with no system-account protection is a resume-generating event.
- **The honest answer:** at any real scale you don't manage users with a bash script. You use Ansible, LDAP/AD, or AWS SSM Session Manager with IAM — no local users at all. Say this. Knowing the limits of the tool is the senior signal.

<!-- related -->
---
**Related:** [[Lab-Server-Health-Check-Script|Lab — Server Health Check Script]]
<!-- /related -->
