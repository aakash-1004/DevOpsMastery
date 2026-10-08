---
title: "Shell Scripting — Real Script, Special Variables, Redirection, /dev/null"
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
order: 6
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-05-Functions-Arguments-and-Useful-Concepts]]"
next: "[[Shell-07-Logging-Debugging-and-Scheduling]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Real Script, Special Variables, Redirection, /dev/null

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-05-Functions-Arguments-and-Useful-Concepts|Shell Scripting — Functions, Arguments, Other Useful Concepts]] · → [[Shell-07-Logging-Debugging-and-Scheduling|Shell Scripting — Logging, Debugging, Scheduling (at, cron)]]
<!-- /nav -->

## 23. Real script — connectivity check

```bash
#!/usr/bin/env bash
#
# check-connectivity.sh — verify a list of hosts is reachable
# Usage: ./check-connectivity.sh hosts.txt
#
set -euo pipefail

HOSTFILE="${1:?Usage: $0 <hostfile>}"      # why: :? gives a clean error if no argument supplied
PING_COUNT=2
TIMEOUT=3
FAILED=0

[[ -f "$HOSTFILE" ]] || { echo "File not found: $HOSTFILE" >&2; exit 1; }

printf "%-25s %-18s %s\n" "HOST" "IP" "STATUS"    # why: printf for aligned columns; echo can't do this
printf "%-25s %-18s %s\n" "-------------------------" "------------------" "------"

while IFS= read -r host || [[ -n "$host" ]]; do   # why: || [[ -n ]] handles a missing trailing newline
  [[ -z "$host" || "$host" =~ ^[[:space:]]*# ]] && continue   # why: skip blanks and comments

  # Resolve DNS first — distinguishes "DNS broken" from "host down"
  if ! ip=$(getent hosts "$host" | awk '{print $1; exit}'); then   # why: getent uses the system resolver (nsswitch), unlike dig
    printf "%-25s %-18s %s\n" "$host" "-" "DNS_FAIL"
    (( FAILED++ ))
    continue
  fi

  if ping -c "$PING_COUNT" -W "$TIMEOUT" "$host" &>/dev/null; then   # why: -W per-packet timeout stops a long hang
    printf "%-25s %-18s %s\n" "$host" "$ip" "UP"
  else
    printf "%-25s %-18s %s\n" "$host" "$ip" "DOWN"
    (( FAILED++ ))
  fi
done < "$HOSTFILE"

echo
if (( FAILED > 0 )); then
  echo "$FAILED host(s) unreachable"
  exit 1                       # why: non-zero exit so CI/monitoring can detect failure
fi
echo "All hosts reachable"
```

### Port-level check (more useful than ping in cloud environments)

```bash
check_port() {
  local host="$1" port="$2" timeout="${3:-3}"
  timeout "$timeout" bash -c "</dev/tcp/${host}/${port}" 2>/dev/null
  # why: /dev/tcp is a bash PSEUDO-DEVICE — opens a TCP socket with no nc/telnet needed.
  # why: critical in minimal containers where nc isn't installed.
}

check_port db.internal 5432 && echo "DB reachable" || echo "DB unreachable"
```

**ICMP is blocked by default in most AWS security groups**, so `ping` failing proves nothing. Always say this if asked how you'd test connectivity to an EC2 instance or RDS endpoint — test the actual port with `/dev/tcp`, `nc -z`, or `curl`.

---

## 24. Special BASH variables **[MUST KNOW]**

```bash
$0          # script name as invoked
$1..$9      # positional parameters (${10}+ need braces)
$#          # number of arguments
$@          # all arguments as separate words (quoted)
$*          # all arguments as one string
$?          # exit code of the LAST command
$$          # PID of the current shell/script — used for unique temp filenames
$!          # PID of the last BACKGROUND process
$_          # last argument of the previous command
$-          # current shell option flags
```

```bash
# Bash-specific
$BASH_VERSION       # bash version string
$BASH_SOURCE        # array; ${BASH_SOURCE[0]} = path of the current script (correct even when sourced, unlike $0)
$LINENO             # current line number — use in error messages
$FUNCNAME           # array of the function call stack; ${FUNCNAME[0]} = current function
$PIPESTATUS         # array of exit codes for EVERY command in the last pipeline
$RANDOM             # random integer 0-32767
$SECONDS            # seconds since the script started — free runtime measurement
$HOSTNAME $USER $HOME $PWD $OLDPWD $PATH $UID $EUID
$IFS                # internal field separator: default space, tab, newline
$PS1 $PS2 $PS4      # prompts; PS4 is the set -x trace prefix
$EPOCHSECONDS       # unix timestamp (bash 5+), no `date` fork needed
```

### `$PIPESTATUS` — underused and impressive

```bash
curl -sf https://api.example.com | jq '.items' | wc -l
echo "${PIPESTATUS[@]}"      # e.g. "0 0 0" — exit code of EACH stage
# why: with pipefail you know SOMETHING failed; PIPESTATUS tells you WHICH stage.
```

### Practical uses

```bash
TMPFILE="/tmp/deploy.$$"                    # why: $$ makes the filename unique per process run
long_task &                                 # why: & backgrounds it
TASK_PID=$!                                 # why: capture its PID so you can wait/kill it later
wait "$TASK_PID"                            # why: block until it finishes and get its exit code

echo "Completed in ${SECONDS}s"             # why: free elapsed-time measurement
echo "ERROR at ${BASH_SOURCE[0]}:${LINENO} in ${FUNCNAME[0]}" >&2   # why: precise error location

export PS4='+ ${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]}: '   # why: makes set -x output show file:line:function
```

### Interview Q&A

**Q: What is `$$` and where would you use it?**

> The PID of the current shell. Its main practical use is building unique temp filenames or PID files so concurrent runs of the same script don't collide — though `mktemp` is the safer choice because it also handles permissions and symlink attacks.

**Q: What's `$PIPESTATUS` for?**

> It's an array of the exit codes of every command in the last pipeline. `$?` only gives you the last stage, and even with `pipefail` you only learn that _something_ failed. `PIPESTATUS` tells you exactly which stage failed, which matters when you're debugging a four-stage pipeline in a CI job.

**Q: `$0` vs `${BASH_SOURCE[0]}`?**

> `$0` is the name the shell was invoked with — when a script is `source`d, `$0` is the _parent_ script's name. `${BASH_SOURCE[0]}` is always the path of the file currently executing, so it's what you use for the `SCRIPT_DIR` idiom and for error messages in sourced libraries.

---

## 25. Redirection **[MUST KNOW]**

```bash
command > file          # why: stdout to file, TRUNCATES existing content
command >> file         # why: stdout appended
command 2> file         # why: stderr only
command 2>> file        # why: stderr appended
command &> file         # why: BOTH stdout+stderr (bash shorthand)
command > file 2>&1     # why: POSIX way — both streams. ORDER MATTERS, see below.
command < file          # why: file as stdin
command << EOF          # why: here-document
command <<< "string"    # why: here-string
command 2>&1 | less     # why: pipe stderr too — a plain pipe only carries stdout
command > /dev/null 2>&1  # why: discard everything
```

### The ordering trap **[classic interview question]**

```bash
command > file 2>&1     # CORRECT: stdout→file, then stderr→wherever stdout now points (file)
command 2>&1 > file     # WRONG:   stderr→current stdout (TERMINAL), then stdout→file. stderr stays on screen.
```

Redirections are processed **left to right**, and `2>&1` copies _where stdout points at that moment_ — it does not create a permanent link.

### File descriptors

```
0 = stdin
1 = stdout
2 = stderr
3+ = custom
```

```bash
echo "error message" >&2            # why: send to stderr — ALWAYS do this for errors
exec 3>&1                           # why: save current stdout into FD 3
exec 1>/var/log/script.log          # why: redirect ALL subsequent stdout in the script to a logfile
echo "goes to log"
exec 1>&3 3>&-                      # why: restore stdout from FD 3 and close FD 3
```

### `tee` — write to file and screen

```bash
command | tee output.log                 # why: display AND save
command | tee -a output.log              # why: append instead of overwrite
command 2>&1 | tee output.log            # why: capture both streams
command | tee /dev/stderr | other_cmd    # why: inspect data mid-pipeline
./deploy.sh 2>&1 | tee -a deploy.log     # why: THE standard pattern for a deploy script
```

**Caution with `tee` and exit codes:** `cmd | tee log` returns `tee`'s exit code, so a failing `cmd` looks successful. Use `set -o pipefail` or check `${PIPESTATUS[0]}`.

### Why stderr separation matters in DevOps

```bash
RESULT=$(get_version)      # why: captures stdout ONLY

get_version() {
  echo "Fetching version..." >&2    # why: log to stderr — does NOT contaminate $RESULT
  echo "v1.2.3"                     # why: actual data on stdout
}
```

This is also why `kubectl get pods 2>/dev/null` is safe in a pipeline — warnings go to stderr, data to stdout.

### Interview Q&A

**Q: Difference between `2>&1 > file` and `> file 2>&1`?**

> Redirections apply left to right. `> file 2>&1` points stdout at the file first, then points stderr at wherever stdout currently is — the file. `2>&1 > file` points stderr at the terminal (stdout's value at that moment), then moves stdout to the file, so stderr still prints to the screen. The correct order is `> file 2>&1`.

**Q: How do you log a script's output to a file and still see it live?**

> `./script.sh 2>&1 | tee -a run.log`. I'd pair it with `set -o pipefail` because `tee` always exits 0 and would otherwise mask a failing script.

---

## 26. /dev/null **[MUST KNOW]**

`/dev/null` is a character device that discards everything written to it and returns EOF immediately on read. Often called the bit bucket.

```bash
command > /dev/null             # why: discard stdout, KEEP stderr visible — usually what you want
command 2> /dev/null            # why: discard stderr only (suppress "permission denied" noise from find)
command > /dev/null 2>&1        # why: discard everything — use only when you check $? afterwards
command &> /dev/null            # why: bash shorthand for the same
command < /dev/null             # why: give the command EMPTY stdin — stops it consuming the loop's input

find / -name "*.conf" 2>/dev/null                    # why: hide permission-denied spam, keep real results
if ping -c1 host &>/dev/null; then ...; fi           # why: only the exit code matters, output is noise
systemctl is-active nginx >/dev/null 2>&1            # why: test-only, suppress all output
```

**Caution:** `> /dev/null 2>&1` hides the error message too. If a cron job does this and fails, you have nothing to debug. Prefer sending errors to a log:

```bash
./job.sh >/dev/null 2>>/var/log/job-errors.log       # why: discard noise, KEEP errors for diagnosis
```

### Interview Q&A

**Q: What's `/dev/null` and when do you use it?**

> A kernel device that discards all writes. I use it when only a command's exit code matters — connectivity checks, `command -v` lookups — or to silence known-irrelevant stderr like find's permission-denied messages. I'm careful not to blanket-silence a cron job, because then a failure leaves no evidence; I'd redirect stderr to a log file instead.
