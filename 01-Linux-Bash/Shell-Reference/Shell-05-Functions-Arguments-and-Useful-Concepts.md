---
title: "Shell Scripting — Functions, Arguments, Other Useful Concepts"
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
order: 5
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-04-Loops]]"
next: "[[Shell-06-Real-Script-Special-Variables-Redirection]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Functions, Arguments, Other Useful Concepts

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-04-Loops|Shell Scripting — Loops]] · → [[Shell-06-Real-Script-Special-Variables-Redirection|Shell Scripting — Real Script, Special Variables, Redirection, /dev/null]]
<!-- /nav -->

## 20. Functions **[MUST KNOW]**

```bash
# POSIX-compatible form — PREFERRED
deploy() {
  local env="$1"              # why: ALWAYS declare local, or you pollute global scope
  local image="${2:-latest}"  # why: default value for optional parameter
  echo "Deploying $image to $env"
}

# bash-only form (works but less portable)
function deploy { ... }

deploy "prod" "v1.2.3"        # why: call with SPACE-separated args, no parentheses, no commas
```

### Arguments inside a function

Inside a function, `$1 $2 $@ $#` refer to the **function's** arguments, not the script's. `$0` still refers to the script name.

```bash
show_args() {
  echo "Count: $#"
  echo "All: $@"
  echo "First: $1"
  echo "Script: $0"          # why: $0 is NOT the function name — it's still the script
  echo "Function: ${FUNCNAME[0]}"   # why: THIS is how you get the function's own name
}
```

### Return values — the key concept **[MUST KNOW]**

```bash
# `return` returns an EXIT CODE (0-255), NOT data.
is_running() {
  systemctl is-active --quiet "$1"   # why: the function's exit code is that of its last command
  return $?                          # why: explicit, though redundant here
}

if is_running nginx; then echo "nginx up"; fi

# To return DATA, echo it and capture with $( ):
get_instance_id() {
  curl -s http://169.254.169.254/latest/meta-data/instance-id   # why: EC2 IMDS endpoint
}
ID=$(get_instance_id)

# Mixing both — data on stdout, status via return:
get_pod_count() {
  local ns="$1"
  local count
  count=$(kubectl get pods -n "$ns" --no-headers 2>/dev/null | wc -l) || return 1
  echo "$count"       # why: data on STDOUT
  return 0            # why: status via return
}
```

**Common mistake:** `return "some string"` — bash only accepts integers 0-255. `return 300` silently becomes 44 (mod 256).

**Important:** any `echo` inside a function is captured by `$(...)`. If you add a debug `echo "checking..."` inside a function whose output you capture, it corrupts the return value. **Send logs to stderr: `echo "checking..." >&2`.**

### Local and recursion

```bash
countdown() {
  local n="$1"                        # why: without `local`, recursion overwrites the caller's n
  (( n <= 0 )) && { echo "Done"; return 0; }
  echo "$n"
  countdown $(( n - 1 ))
}
```

### Function libraries — real DevOps pattern

```bash
# lib/common.sh
log_info()  { echo "[$(date +'%F %T')] [INFO]  $*" >&2; }    # why: logs to stderr so they don't pollute captured stdout
log_error() { echo "[$(date +'%F %T')] [ERROR] $*" >&2; }
die()       { log_error "$*"; exit 1; }                       # why: log-and-exit in one call
require()   { command -v "$1" >/dev/null || die "$1 not installed"; }
```

```bash
# main.sh
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"   # why: absolute dir of THIS script, works from any cwd
source "${SCRIPT_DIR}/lib/common.sh"                          # why: import shared functions

require kubectl
require aws
log_info "Starting deployment"
```

That `SCRIPT_DIR` line is worth memorising — it's how you make a script's relative paths work regardless of where it's invoked from, which is exactly what breaks scripts under cron.

### Interview Q&A

**Q: How do you return a value from a bash function?**

> `return` only sets an exit status, 0-255 — that's a status, not data. To return data I `echo` it and capture with command substitution. If the function also needs to log, those logs go to stderr so they don't get mixed into the captured value.

**Q: Why use `local` in functions?**

> Bash variables are global by default, so a function assigning to `i` or `count` silently overwrites the caller's variable. `local` scopes it to the function and its callees. Without it, nested loops and recursion break in ways that are very hard to debug.

**Q: How do you reuse functions across scripts?**

> Put them in a library file and `source` it. I resolve the path with `SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"` so sourcing works no matter what directory the script is invoked from — important under cron, where the working directory is the user's home.

---

## 21. Passing arguments to a script **[MUST KNOW]**

```bash
./deploy.sh prod v1.2.3 --force
```

```bash
echo "$0"      # ./deploy.sh — script name/path as invoked
echo "$1"      # prod
echo "$2"      # v1.2.3
echo "$#"      # 3  — number of arguments
echo "$@"      # all args as SEPARATE words (when quoted)
echo "$*"      # all args as ONE string joined by first char of IFS
echo "${10}"   # why: BRACES REQUIRED for args 10+; $10 is parsed as $1 followed by "0"
```

### `"$@"` vs `"$*"` — guaranteed interview question

```bash
# Called as: ./s.sh "hello world" foo
for a in "$@"; do echo "[$a]"; done   # [hello world] [foo]   ← 2 args preserved
for a in "$*"; do echo "[$a]"; done   # [hello world foo]     ← 1 merged string
for a in $@;   do echo "[$a]"; done   # [hello] [world] [foo] ← UNQUOTED, broken
```

**Always `"$@"` when forwarding arguments to another command.**

### shift

```bash
while (( $# > 0 )); do
  echo "Processing: $1"
  shift                    # why: discards $1 and shifts everything left — $2 becomes $1, $# decreases
done
```

### Argument validation — the part that matters

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<EOF                                    # why: here-doc is the clean way to print multi-line usage text
Usage: $(basename "$0") <environment> <version> [--dry-run]

Arguments:
  environment   dev | staging | prod
  version       image tag, e.g. v1.2.3

Options:
  --dry-run     print actions without executing
EOF
  exit 1
}

(( $# >= 2 )) || usage                         # why: fail early with usage text if required args missing

ENVIRONMENT="$1"
VERSION="$2"
DRY_RUN=false
shift 2                                        # why: consume the positional args so the flag loop sees only flags

while (( $# > 0 )); do
  case "$1" in
    --dry-run) DRY_RUN=true; shift ;;
    -h|--help) usage ;;
    *) echo "Unknown option: $1" >&2; usage ;;
  esac
done

case "$ENVIRONMENT" in
  dev|staging|prod) ;;                         # why: whitelist valid values instead of blacklisting bad ones
  *) echo "Invalid environment: $ENVIRONMENT" >&2; exit 1 ;;
esac

[[ "$VERSION" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || { echo "Invalid version format" >&2; exit 1; }
```

### getopts — for proper flag parsing **[GOOD TO KNOW]**

```bash
while getopts ":e:v:dh" opt; do    # why: leading : = silent error mode; letter: = that flag takes an argument
  case "$opt" in
    e) ENVIRONMENT="$OPTARG" ;;    # why: OPTARG holds the flag's value
    v) VERSION="$OPTARG" ;;
    d) DRY_RUN=true ;;
    h) usage ;;
    \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
    :)  echo "Option -$OPTARG requires an argument" >&2; exit 1 ;;
  esac
done
shift $(( OPTIND - 1 ))            # why: remove parsed flags, leaving only positional args in $@
```

`getopts` handles short flags only (`-e prod`). It does **not** handle GNU long options (`--env=prod`) — for those you write a manual `case` loop like the one above. Knowing that distinction is a good signal in an interview.

### Interview Q&A

**Q: `$@` vs `$*`?**

> Quoted, `"$@"` expands to each argument as a separate word, preserving arguments that contain spaces. `"$*"` joins everything into one string using `IFS`. When forwarding arguments to another command you always want `"$@"` — `"$*"` would collapse `"my file.txt"` into separate words downstream.

**Q: How do you handle optional flags in a bash script?**

> `getopts` for short flags — it handles bundling and missing-argument errors for you. For long options I write a `while (( $# ))` loop with a `case` on `$1` and `shift`. Either way I validate required arguments up front and print a usage block from a here-doc, so the failure message tells the caller exactly what's wrong.

---

## 22. Other useful concepts **[MUST KNOW — this section is where senior candidates separate themselves]**

### `set` options — put these in every script

```bash
set -e                 # why: exit immediately if any command returns non-zero
set -u                 # why: treat unset variables as an error — catches typos like $DEPLOY_ENVV
set -o pipefail        # why: a pipeline's exit code = first FAILING command, not the last one
set -x                 # why: print each command before executing (debug only, never in prod with secrets)

set -euo pipefail      # why: THE standard first line of every production bash script
IFS=$'\n\t'            # why: optional hardening — removes space from IFS so filenames with spaces don't split
```

**Why `pipefail` matters — ask yourself what this returns:**

```bash
false | true; echo $?           # 0  — the failure was silently swallowed
set -o pipefail
false | true; echo $?           # 1  — correct

# Real impact:
curl -sf https://bad-url | jq '.version'   # without pipefail, a failed curl still "succeeds" because jq exits 0
```

**Limits of `set -e` you should be able to name:**

- Doesn't trigger for commands in `if`, `while`, `&&`, `||`, or `!` conditions
- Doesn't trigger inside command substitution in older bash versions
- Doesn't trigger for the left side of a pipe without `pipefail`
- A function that fails inside an `if` won't exit the script

Because of these, `set -e` is a safety net, not a substitute for checking exit codes on anything critical.

### Exit codes **[MUST KNOW]**

```bash
exit 0      # success
exit 1      # generic failure
exit 2      # misuse of shell builtin / bad arguments (convention)
exit 126    # command found but not executable
exit 127    # command not found
exit 128+N  # killed by signal N  →  130 = SIGINT (Ctrl+C), 137 = SIGKILL (OOMKilled!), 143 = SIGTERM
```

**137 is worth memorising**: `128 + 9` = SIGKILL. In Kubernetes, a container with exit code 137 was OOMKilled or force-terminated. **143** = `128 + 15` = SIGTERM, i.e. a graceful shutdown that didn't finish in time. Interviewers love this one.

```bash
command
echo "$?"             # why: $? holds the exit code of the LAST command — check it IMMEDIATELY, it's overwritten by the next command

# WRONG:
some_command
echo "Running check"   # why: this echo just reset $?
if [[ $? -ne 0 ]]; then ... fi   # now checking echo's exit code, not some_command's

# RIGHT:
if ! some_command; then ... fi
```

### `trap` — cleanup and signal handling **[MUST KNOW for production scripts]**

```bash
TMPDIR=$(mktemp -d)                    # why: mktemp -d creates a secure unique temp dir; never hardcode /tmp/mydir

cleanup() {
  local rc=$?                          # why: capture the exit code BEFORE doing anything else
  rm -rf "$TMPDIR"
  echo "Cleaned up, exiting with $rc"
  exit "$rc"                           # why: preserve the original exit code
}

trap cleanup EXIT                      # why: EXIT fires on ANY exit — normal, error, or signal. The one to always set.
trap 'echo "Interrupted"; exit 130' INT TERM   # why: handle Ctrl+C and kill gracefully
trap 'echo "Error on line $LINENO"' ERR        # why: fires on any command returning non-zero (works with set -e)
```

**Where this matters:** a deploy script that creates a temp kubeconfig, or a backup script that takes a lock. Without `trap ... EXIT`, a Ctrl+C or a mid-script failure leaves garbage behind. In a container entrypoint, trapping SIGTERM is how you implement graceful shutdown during a rolling update.

### Here-documents and here-strings **[MUST KNOW]**

```bash
cat <<EOF > /etc/app/config.yaml      # why: variables ARE expanded with unquoted EOF
environment: $ENVIRONMENT
replicas: $REPLICAS
EOF

cat <<'EOF' > script.sh               # why: QUOTED 'EOF' = NO expansion — essential when writing scripts/awk/$ literals
#!/bin/bash
echo "$HOME stays literal"
EOF

cat <<-EOF                            # why: <<- strips LEADING TABS so you can indent the heredoc inside a function
	indented in source, flush in output
	EOF

grep "x" <<< "$STRING"                # why: <<< here-string feeds a single string as stdin — avoids echo | grep
```

**The quoted-vs-unquoted delimiter distinction is a real interview question**, and it's how you avoid mangling a generated script.

### Process substitution **[GOOD TO KNOW — scores well]**

```bash
diff <(kubectl get cm app -o yaml) <(kubectl get cm app -n staging -o yaml)
# why: <(cmd) presents a command's output as a FILE — lets you diff two commands without temp files

while read -r line; do ... done < <(kubectl get pods)   # why: avoids the subshell problem a pipe would cause

tee >(gzip > out.gz) >(wc -l > count.txt) > /dev/null   # why: >(cmd) is the output direction
```

### Subshells and grouping

```bash
( cd /tmp && ./script.sh )     # why: PARENTHESES = subshell — the cd does NOT affect the parent shell
{ cd /tmp && ./script.sh; }    # why: BRACES = current shell — cd DOES persist. Note the required trailing ;

VAR=1
( VAR=2 ); echo "$VAR"         # 1 — subshell changes are discarded
{ VAR=2; }; echo "$VAR"        # 2
```

### `xargs`

```bash
find /var/log -name "*.log" -mtime +30 | xargs rm              # why: breaks on filenames with spaces
find /var/log -name "*.log" -mtime +30 -print0 | xargs -0 rm   # why: -print0/-0 use NUL separator — SAFE
find /var/log -name "*.log" -mtime +30 -delete                 # why: simplest when find supports it

kubectl get pods -o name | xargs -I{} kubectl logs {}          # why: -I{} substitutes each line into the placeholder
cat urls.txt | xargs -P 4 -n 1 curl -s                         # why: -P 4 = 4 parallel processes, -n 1 = one arg each
```

### Idempotency **[MUST KNOW — asked in every automation interview]**

Running the script twice must produce the same result as running it once.

```bash
mkdir -p /opt/app                                   # why: -p doesn't error if it already exists
ln -sfn /opt/app/v2 /opt/app/current                # why: -f replace, -n don't follow existing symlink
grep -q "^entry$" file || echo "entry" >> file      # why: check before append — prevents duplicate lines
useradd deploy 2>/dev/null || true                  # why: tolerate "already exists"
systemctl enable --now nginx                        # why: enable+start is naturally idempotent
```

### Locking — preventing concurrent runs **[GOOD TO KNOW]**

```bash
LOCKFILE="/var/lock/backup.lock"
exec 200>"$LOCKFILE"                       # why: open FD 200 on the lockfile for the script's lifetime
flock -n 200 || { echo "Already running"; exit 1; }   # why: -n = non-blocking, fail instead of waiting
# ... critical work ...
# lock releases automatically when the script exits and FD 200 closes
```

A cron job that takes longer than its interval will otherwise stack up and eventually exhaust the box. `flock` is the standard fix and it's a strong answer when asked "how do you prevent overlapping cron runs?"

### Secrets handling in scripts **[MUST KNOW for DevOps]**

```bash
# NEVER:
mysql -u root -p"$PASSWORD"          # why: visible in `ps aux` to every user on the box
export AWS_SECRET_ACCESS_KEY="AKI…"  # why: ends up in /proc/<pid>/environ and in `set -x` output
echo "$TOKEN" >> deploy.log          # why: secrets in logs = secrets in your log aggregator forever

# DO:
mysql --defaults-extra-file=/root/.my.cnf     # why: credentials in a chmod 600 file, not on the command line
aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text
set +x                                        # why: disable tracing around the secret-handling block
# ... handle secret ...
set -x
docker run --env-file ./secrets.env image     # why: --env-file (chmod 600), NOT -e KEY=value on the CLI
```

### Interview Q&A

**Q: What's the first line you put in every bash script and why?**

> `set -euo pipefail`. `-e` exits on any failing command so errors don't cascade, `-u` turns unset-variable typos into hard errors instead of silently expanding to empty strings, and `pipefail` makes a pipeline fail if any stage fails instead of only the last one. Without pipefail, something like `curl bad-url | jq .` reports success.

**Q: What does exit code 137 mean?**

> 128 plus signal 9, so the process was SIGKILLed. In Kubernetes that almost always means OOMKilled — the container exceeded its memory limit. I'd confirm with `kubectl describe pod` and check `Last State: Terminated, Reason: OOMKilled`, then decide between raising the limit and fixing the memory usage.

**Q: How do you make sure a script cleans up after itself even if it's interrupted?**

> `trap cleanup EXIT`. The EXIT trap fires on normal exit, on `set -e` failures, and on signals that terminate the shell. I capture `$?` as the first line of the cleanup function so the original exit code is preserved after cleanup runs.

**Q: How do you stop two copies of a cron job running at once?**

> `flock` on a lockfile with `-n` so the second instance exits immediately instead of queueing. The lock releases automatically when the file descriptor closes on exit, so there's no stale-lock cleanup to get wrong.
