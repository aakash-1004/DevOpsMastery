---
title: Shell Scripting — Loops
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
order: 4
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-03-Conditionals-and-Logic]]"
next: "[[Shell-05-Functions-Arguments-and-Useful-Concepts]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Loops

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-03-Conditionals-and-Logic|Shell Scripting — Conditionals, case, Logical Operators]] · → [[Shell-05-Functions-Arguments-and-Useful-Concepts|Shell Scripting — Functions, Arguments, Other Useful Concepts]]
<!-- /nav -->

## 18. For loop **[MUST KNOW]**

```bash
# ---- list ----
for env in dev staging prod; do
  echo "Deploying to $env"
done

# ---- array ----
for server in "${SERVERS[@]}"; do     # why: ALWAYS quote "${arr[@]}"
  ssh "$server" uptime
done

# ---- C-style ----
for (( i=1; i<=5; i++ )); do
  echo "Attempt $i"
done

# ---- range / brace expansion ----
for i in {1..10}; do echo "$i"; done         # why: literal brace expansion, no subprocess
for i in {0..20..5}; do echo "$i"; done      # why: step of 5 (bash 4+)
for i in $(seq 1 10); do echo "$i"; done     # why: seq forks a process; brace expansion is better
for i in $(seq 1 "$N"); do ... done          # why: BUT brace expansion can't use variables — {1..$N} fails

# ---- files (glob) ----
for f in /var/log/*.log; do
  [[ -e "$f" ]] || continue                  # why: if no match, glob stays literal "*.log" — this guards it
  echo "Processing $f"
done

# ---- command output ----
for pod in $(kubectl get pods -o name); do   # why: fine HERE because pod names have no spaces
  echo "$pod"
done
```

### For loop over a file — the right and wrong way **[MUST KNOW]**

```bash
# WRONG — splits on spaces, not lines:
for line in $(cat servers.txt); do echo "$line"; done

# WRONG-ish — fixing IFS works but is fragile:
IFS=$'\n'
for line in $(cat servers.txt); do echo "$line"; done

# CORRECT — use while read:
while IFS= read -r line; do          # why: IFS= preserves leading/trailing whitespace; -r keeps backslashes literal
  echo "Processing: $line"
done < servers.txt                    # why: redirect the FILE into the loop, not cat into a pipe
```

**This is a very common interview question: "read a file line by line in bash."** The answer is `while IFS= read -r line; do ... done < file`, and you should be able to explain every part.

### Loop control

```bash
for i in {1..10}; do
  [[ $i -eq 3 ]] && continue    # why: skip this iteration
  [[ $i -eq 7 ]] && break       # why: exit the loop entirely
  echo "$i"
done

for i in {1..3}; do
  for j in {1..3}; do
    [[ $j -eq 2 ]] && break 2   # why: break out of TWO levels of loop
  done
done
```

### Interview Q&A

**Q: Why is `for line in $(cat file)` wrong?**

> It word-splits on all of IFS — spaces, tabs, newlines — not just newlines. A line like `web01 production` becomes two iterations. It also glob-expands, so a line containing `*` gets replaced by filenames. The correct pattern is `while IFS= read -r line; do ... done < file`.

**Q: Why `{1..10}` over `$(seq 1 10)`?**

> Brace expansion is handled by the shell itself with no subprocess. `seq` forks an external binary each time. The trade-off is that brace expansion can't take variables — `{1..$N}` doesn't expand — so for a variable count I use a C-style loop `for (( i=1; i<=N; i++ ))`.

---

## 19. While, until, and infinite loops **[MUST KNOW]**

### while

```bash
COUNT=1
while (( COUNT <= 5 )); do
  echo "Count: $COUNT"
  (( COUNT++ ))
done
```

### until — runs _until_ the condition becomes true

```bash
until curl -sf http://localhost:8080/health >/dev/null; do   # why: -s silent, -f fail on HTTP errors
  echo "Waiting for app to be ready..."
  sleep 2
done
echo "App is up"
```

**`until` is the natural fit for wait-for-readiness logic** — container entrypoints, CI steps waiting on a service. That's its main real-world use.

### Infinite loops

```bash
while true; do ... ; sleep 5; done     # why: most readable, what you'll see in real scripts
while :; do ... ; done                 # why: : is the null builtin that always returns 0 — marginally faster
for (( ; ; )); do ... ; done           # why: C-style infinite
```

**Always put a `sleep` in an infinite loop.** A tight loop with no sleep will peg a CPU core at 100%. This is a real production incident waiting to happen and interviewers do ask about it.

### Wait-with-timeout pattern **[MUST KNOW for container/CI work]**

```bash
TIMEOUT=60
ELAPSED=0
INTERVAL=3

until nc -z db.internal 5432 2>/dev/null; do     # why: nc -z tests TCP port without sending data
  if (( ELAPSED >= TIMEOUT )); then
    echo "ERROR: database not reachable after ${TIMEOUT}s" >&2
    exit 1                                        # why: ALWAYS bound the wait — unbounded waits hang CI pipelines forever
  fi
  echo "Waiting for database... (${ELAPSED}s)"
  sleep "$INTERVAL"
  (( ELAPSED += INTERVAL ))
done
echo "Database reachable"
```

### Retry with exponential backoff **[GOOD TO KNOW — scores well]**

```bash
retry() {
  local max=5 delay=1 attempt=1
  until "$@"; do                                  # why: "$@" runs whatever command was passed in
    if (( attempt >= max )); then
      echo "Failed after $max attempts: $*" >&2
      return 1
    fi
    echo "Attempt $attempt failed, retrying in ${delay}s..." >&2
    sleep "$delay"
    (( delay *= 2 ))                              # why: exponential backoff avoids hammering a struggling service
    (( attempt++ ))
  done
}

retry curl -sf https://api.example.com/health
```

### While loop with a file **[MUST KNOW]**

```bash
while IFS= read -r line; do
  echo "Line: $line"
done < input.txt

# Parsing structured lines with multiple fields:
while IFS=, read -r name ip env; do               # why: IFS=, splits each line on commas into 3 variables
  echo "$name ($ip) in $env"
done < servers.csv

# Skip comments and blank lines (standard config-file pattern):
while IFS= read -r line; do
  [[ -z "$line" || "$line" =~ ^[[:space:]]*# ]] && continue   # why: ignore empties and # comments
  echo "Config: $line"
done < app.conf

# Handle a file with NO trailing newline (last line gets silently dropped otherwise):
while IFS= read -r line || [[ -n "$line" ]]; do   # why: the || clause catches the final unterminated line
  echo "$line"
done < input.txt
```

### The subshell trap with pipes **[MUST KNOW — classic gotcha]**

```bash
COUNT=0
cat file.txt | while read -r line; do
  (( COUNT++ ))
done
echo "$COUNT"      # prints 0 !!

# Why: each side of a pipe runs in a SUBSHELL. COUNT++ happened in the child and died with it.

# FIX 1 — redirect instead of pipe:
COUNT=0
while read -r line; do (( COUNT++ )); done < file.txt
echo "$COUNT"      # correct

# FIX 2 — process substitution (needed when the input is a command, not a file):
COUNT=0
while read -r line; do (( COUNT++ )); done < <(kubectl get pods --no-headers)
echo "$COUNT"      # correct

# FIX 3 — shopt (bash 4.2+):
shopt -s lastpipe  # why: runs the LAST pipeline command in the current shell (requires job control off)
```

### The SSH-eats-stdin trap **[GOOD TO KNOW — very real]**

```bash
# BROKEN — ssh consumes the rest of the loop's stdin, loop runs once:
while read -r host; do
  ssh "$host" uptime
done < servers.txt

# FIX:
while read -r host; do
  ssh -n "$host" uptime       # why: -n redirects ssh's stdin from /dev/null so it stops eating the file
done < servers.txt

# Same class of bug with other stdin-reading commands — use < /dev/null:
while read -r f; do
  some_command < /dev/null
done < list.txt
```

### Interview Q&A

**Q: Why does a counter incremented inside `cat file | while read` end up as 0?**

> Both sides of a pipeline run in subshells. The increment happens in the child process, and when the pipeline ends the child's variables are discarded. I avoid it by redirecting the file directly — `while read ... done < file` — or using process substitution `< <(command)` when the input is a command's output.

**Q: Your loop over a host list only processes the first host. What happened?**

> Something inside the loop is consuming stdin — almost always `ssh`. Add `-n` to ssh, or redirect that command's stdin from `/dev/null`.

**Q: `while` vs `until`?**

> `while` loops while the condition is true, `until` loops until it becomes true. Functionally `until cond` equals `while ! cond`. I use `until` for readiness waits because "until the service is healthy" reads naturally.
