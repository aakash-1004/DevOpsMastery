---
title: "Shell Scripting — Conditionals, case, Logical Operators"
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
order: 3
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-02-Variables-Arrays-Strings-Input-Arithmetic]]"
next: "[[Shell-04-Loops]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Conditionals, case, Logical Operators

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-02-Variables-Arrays-Strings-Input-Arithmetic|Shell Scripting — Variables, Arrays, Strings, Input, Arithmetic]] · → [[Shell-04-Loops|Shell Scripting — Loops]]
<!-- /nav -->

## 14. If-else and test conditions **[MUST KNOW]**

```bash
if [[ condition ]]; then
  # ...
else
  # ...
fi
```

### `[ ]` vs `[[ ]]` vs `(( ))` — guaranteed interview question

||`[ ]` / `test`|`[[ ]]`|`(( ))`|
|---|---|---|---|
|Type|External-style builtin, POSIX|Bash keyword|Bash arithmetic keyword|
|Unquoted empty var|**Breaks** (syntax error)|Safe|Safe|
|`&&` / `\|` inside|Not allowed (use `-a`/`-o`)|**Allowed**|Allowed|
|Glob matching `==`|No|**Yes**|No|
|Regex `=~`|No|**Yes**|No|
|Numeric operators|`-gt -lt -eq`|`-gt -lt -eq`|`> < ==`|
|Portability|sh, dash, bash|**bash/ksh/zsh only**|bash only|

**Rule: use `[[ ]]` for strings/files, `(( ))` for numbers, `[ ]` only when writing POSIX sh.**

```bash
VAR=""
[ $VAR = "x" ]      # why: ERROR — expands to `[ = "x" ]`, broken syntax
[[ $VAR == "x" ]]   # why: SAFE — [[ ]] doesn't word-split
```

### File test operators **[MUST KNOW — used constantly]**

```bash
[[ -f "$FILE" ]]       # why: exists AND is a regular file
[[ -d "$DIR" ]]        # why: exists AND is a directory
[[ -e "$PATH" ]]       # why: exists (any type — file, dir, socket, device)
[[ -L "$PATH" ]]       # why: is a symbolic link
[[ -r "$FILE" ]]       # why: readable by current user
[[ -w "$FILE" ]]       # why: writable
[[ -x "$FILE" ]]       # why: executable — use before running a script you didn't create
[[ -s "$FILE" ]]       # why: exists AND size > 0 — catches "backup created but empty" bugs
[[ ! -f "$FILE" ]]     # why: negation
[[ "$F1" -nt "$F2" ]]  # why: F1 newer than F2 (mtime) — useful for rebuild/cache logic
[[ "$F1" -ot "$F2" ]]  # why: older than
```

### String test operators

```bash
[[ "$A" == "$B" ]]     # why: equal (== and = are identical in [[ ]])
[[ "$A" != "$B" ]]     # why: not equal
[[ -z "$A" ]]          # why: zero length / empty
[[ -n "$A" ]]          # why: non-empty
[[ "$A" < "$B" ]]      # why: lexicographic less-than (inside [[ ]] only; in [ ] this is redirection!)
[[ "$A" == dev* ]]     # why: glob match — pattern UNQUOTED
[[ "$A" =~ ^v[0-9]+$ ]]  # why: regex match
```

### Numeric test operators

```bash
[[ $A -eq $B ]]   # equal
[[ $A -ne $B ]]   # not equal
[[ $A -gt $B ]]   # greater than
[[ $A -ge $B ]]   # greater or equal
[[ $A -lt $B ]]   # less than
[[ $A -le $B ]]   # less or equal

(( A > B ))       # why: cleaner equivalent — prefer this for pure arithmetic
```

### Real-world example

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG="/etc/app/config.yaml"

if [[ ! -f "$CONFIG" ]]; then
  echo "ERROR: config not found at $CONFIG" >&2   # why: errors go to stderr so they aren't swallowed by a pipe
  exit 1
elif [[ ! -s "$CONFIG" ]]; then
  echo "ERROR: config exists but is empty" >&2    # why: -s catches the half-written-file case
  exit 1
elif [[ ! -r "$CONFIG" ]]; then
  echo "ERROR: config not readable by $(whoami)" >&2
  exit 1
fi

echo "Config OK"
```

### Interview Q&A

**Q: `[` vs `[[`?**

> `[` is the POSIX test command — it's a builtin but behaves like an external command, so unquoted empty variables break it syntactically and it can't do glob or regex matching. `[[` is a bash keyword, so the shell parses it specially: no word splitting, `&&`/`||` work inside, `==` does glob matching, `=~` does regex. I use `[[ ]]` unless the script must run under POSIX sh.

**Q: Why do people write `[ "x$VAR" = "xvalue" ]`?**

> It's a legacy guard against `[` breaking when `$VAR` is empty or starts with a dash. `[[ ]]` makes it unnecessary, but you'll still see it in old POSIX scripts.

---

## 15. elif **[MUST KNOW]**

```bash
read -rp "Enter CPU usage: " CPU

if (( CPU >= 90 )); then
  echo "CRITICAL"
elif (( CPU >= 70 )); then
  echo "WARNING"
elif (( CPU >= 50 )); then
  echo "ELEVATED"
else
  echo "NORMAL"
fi
```

Order matters — conditions evaluate top to bottom and stop at the first match. Putting `>= 50` first would mask everything below it. Interviewers sometimes hand you an if-chain with the conditions in the wrong order and ask why the alert never fires.

---

## 16. case **[MUST KNOW]**

```bash
case "$1" in
  start)
    echo "Starting service"
    ;;                           # why: ;; terminates the branch — forgetting it is a syntax error
  stop|shutdown)                 # why: | = OR between patterns
    echo "Stopping service"
    ;;
  restart)
    echo "Restarting"
    ;;
  status)
    echo "Checking status"
    ;;
  *)                             # why: default/catch-all — ALWAYS include it
    echo "Usage: $0 {start|stop|restart|status}" >&2
    exit 1
    ;;
esac
```

### Pattern matching in case

```bash
case "$FILE" in
  *.tar.gz|*.tgz)  tar -xzf "$FILE" ;;    # why: glob patterns, not regex
  *.tar.bz2)       tar -xjf "$FILE" ;;
  *.zip)           unzip "$FILE" ;;
  *.log)           gzip "$FILE" ;;
  *)               echo "Unknown type" >&2; exit 1 ;;
esac

case "$ENVIRONMENT" in
  dev|development) REPLICAS=1 ;;
  stag*)           REPLICAS=2 ;;          # why: prefix glob matches staging, stage, stag
  prod|production) REPLICAS=5 ;;
  *)               echo "Unknown env"; exit 1 ;;
esac
```

### Fallthrough (bash 4+)

```bash
case "$x" in
  a) echo "a" ;&      # why: ;& fall THROUGH to next branch unconditionally
  b) echo "b" ;;&     # why: ;;& continue TESTING subsequent patterns
  c) echo "c" ;;
esac
```

### case vs if — when to use which

Use `case` when branching on **one variable against many values** — it's more readable and bash optimises it. Use `if/elif` when each branch tests a **different condition** (ranges, file tests, compound logic). An if-chain with five `[[ "$x" == ... ]]` comparisons should be a `case`.

### Interview Q&A

**Q: When would you choose `case` over `if/elif`?**

> When I'm matching a single variable against a set of known values — service actions, file extensions, environment names. It reads better, supports glob patterns natively, and avoids repeating the variable in every branch. `if/elif` is for different conditions per branch, especially numeric ranges.

---

## 17. Logical operators **[MUST KNOW]**

```bash
cmd1 && cmd2           # why: run cmd2 ONLY IF cmd1 succeeded (exit 0)
cmd1 || cmd2           # why: run cmd2 ONLY IF cmd1 failed (exit non-zero)
! cmd                  # why: negate the exit status
cmd1 ; cmd2            # why: run both regardless of outcome

[[ $A -gt 5 && $B -lt 10 ]]    # why: AND inside [[ ]]
[[ $A -eq 1 || $B -eq 2 ]]     # why: OR inside [[ ]]
[ $A -gt 5 -a $B -lt 10 ]      # why: legacy -a/-o for [ ] — deprecated, avoid
```

**Short-circuit evaluation:** bash stops as soon as the result is determined. `false && anything` never runs `anything`.

### Practical patterns you'll actually write

```bash
mkdir -p /opt/app && cd /opt/app                 # why: only cd if mkdir succeeded

command -v docker >/dev/null 2>&1 || {           # why: command -v is the portable "is it installed?" test
  echo "docker not installed" >&2
  exit 1
}

[[ -f "$CONFIG" ]] || { echo "missing config" >&2; exit 1; }   # why: guard clause idiom

systemctl is-active --quiet nginx && echo "UP" || echo "DOWN"  # why: ternary-ish one-liner
```

### The `&&`/`||` trap with `set -e` **[GOOD TO KNOW — catches senior candidates out]**

```bash
# DANGEROUS pattern:
check_thing && echo "ok" || echo "fail"
# If `echo "ok"` ever fails, `echo "fail"` ALSO runs. The || binds to the whole && chain, not just check_thing.

# Also: a command on the left of && is NOT subject to `set -e`.
set -e
false && echo "x"     # why: script does NOT exit here — false is in a tested context
false                 # why: script DOES exit here
```

For anything non-trivial use an explicit `if`:

```bash
if check_thing; then echo "ok"; else echo "fail"; fi
```

### Interview Q&A

**Q: What does `cmd1 && cmd2 || cmd3` do, and why is it risky?**

> cmd2 runs if cmd1 succeeds, cmd3 runs if the preceding chain fails. The risk is that `||` evaluates the result of the whole `cmd1 && cmd2` chain — so if cmd1 succeeds but cmd2 fails, cmd3 also runs. It's not a real ternary. For anything where cmd2 can fail, I use an explicit if/else.

**Q: How do you check if a command exists before using it?**

> `command -v docker >/dev/null 2>&1 || { echo "missing"; exit 1; }`. `command -v` is a POSIX builtin and handles aliases/functions correctly. `which` is an external binary that isn't present on all images and has inconsistent exit codes.
