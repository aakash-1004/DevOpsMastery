---
title: "Shell Scripting — Variables, Arrays, Strings, Input, Arithmetic"
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
order: 2
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-01-Basics-Shells-Scripts-Shebang]]"
next: "[[Shell-03-Conditionals-and-Logic]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Variables, Arrays, Strings, Input, Arithmetic

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-01-Basics-Shells-Scripts-Shebang|Shell Scripting — Basics: Shells, First Script, Shebang, Running]] · → [[Shell-03-Conditionals-and-Logic|Shell Scripting — Conditionals, case, Logical Operators]]
<!-- /nav -->

## 9. Variables **[MUST KNOW]**

```bash
NAME="aakash"          # why: NO spaces around = — `NAME = "x"` tries to run a command called NAME
echo "$NAME"           # why: ALWAYS quote expansions; unquoted breaks on spaces/globs
echo "${NAME}_suffix"  # why: braces delimit the name so _suffix isn't read as part of it

readonly PI=3.14       # why: constant — any later assignment fails
unset NAME             # why: delete the variable entirely
declare -i COUNT=5     # why: typed as integer; COUNT="abc" becomes 0 instead of a string
declare -r API="x"     # why: same as readonly
declare -x PATH_EXT="/opt/bin"   # why: declare + export in one step
local tmp="x"          # why: function-scoped only — valid ONLY inside a function
```

### Variable scope

```bash
VAR="global"

f() {
  local VAR="local"    # why: without `local`, this OVERWRITES the global — the #1 bash bug in long scripts
  echo "$VAR"
}
f          # local
echo $VAR  # global
```

Bash has no block scope — only global and function-local. **Always `local` inside functions.**

### Environment vs shell variables **[MUST KNOW]**

```bash
MYVAR="hello"          # why: shell variable — visible to this shell only
export MYVAR           # why: now exported to the ENVIRONMENT — inherited by all child processes
env | grep MYVAR       # why: env lists exported variables only
set | grep MYVAR       # why: set lists ALL shell variables + functions
printenv MYVAR         # why: same as env but targeted
```

Children inherit from parents. **Children can never modify the parent's environment.** That's why `export` in a script doesn't affect your terminal.

### Command substitution

```bash
TODAY=$(date +%F)              # why: PREFERRED — nests cleanly, readable
TODAY=`date +%F`               # why: legacy backticks — don't nest, hard to read, avoid in new code
FILES=$(ls -1 | wc -l)
PODS=$(kubectl get pods -o name | wc -l)

# Nesting proves why $() wins:
echo "$(basename "$(dirname "$PWD")")"
```

### Parameter expansion — defaults and guards **[MUST KNOW for production scripts]**

```bash
echo "${VAR:-default}"       # why: use "default" IF VAR unset/empty — does NOT assign
echo "${VAR:=default}"       # why: use AND assign default to VAR
echo "${VAR:?must be set}"   # why: EXIT with this error if VAR unset — best input validation one-liner
echo "${VAR:+alt}"           # why: use "alt" only IF VAR IS set (inverse of :-)

# Real usage in a container entrypoint:
PORT="${PORT:-8080}"                       # why: sane default when env var not injected
ENVIRONMENT="${ENVIRONMENT:?ENVIRONMENT is required}"   # why: fail fast on misconfiguration
AWS_REGION="${AWS_REGION:-ap-south-1}"
```

### Quoting rules — the most common source of bugs **[MUST KNOW]**

```bash
NAME="Aakash Rao"
echo $NAME             # why: WRONG — word-splits into two arguments
echo "$NAME"           # why: CORRECT — one argument
echo '$NAME'           # why: single quotes = literal, NO expansion at all
echo "Path: \$HOME"    # why: backslash escapes the $ inside double quotes

rm $FILE               # why: DANGEROUS — if FILE="" this becomes `rm` with no args; if FILE="a b" it deletes two files
rm -- "$FILE"          # why: CORRECT — quoted, and -- stops filenames starting with - being read as options
```

**Rule: quote every expansion unless you have a specific reason not to.** The only common exception is deliberate word splitting, which you should avoid anyway.

### Interview Q&A

**Q: Difference between shell variable and environment variable?**

> A shell variable lives only in the current shell. An environment variable has been `export`ed, so it's copied into the environment block of every child process the shell spawns. `set` shows both, `env` shows only exported ones. A child can read but never write back to the parent's environment — which is why `export` inside a script has no effect on your terminal after it finishes.

**Q: `$VAR` vs `${VAR}` vs `"$VAR"`?**

> `$VAR` is bare expansion, subject to word splitting and globbing. `${VAR}` is the same but with explicit delimiting, needed when the next character could be part of the name or when you're using parameter expansion. `"$VAR"` prevents word splitting and glob expansion, which is what you want 99% of the time.

**Q: How do you give a variable a default value?**

> `${VAR:-default}` for a fallback, `${VAR:=default}` to also assign it, and `${VAR:?error message}` to hard-fail if it's missing. I use the last one at the top of any script that reads required config from the environment — it turns a confusing downstream failure into a clear one-line error.

---

## 10. Arrays **[MUST KNOW]**

Arrays are **bash-only**. No arrays in dash/POSIX sh.

### Indexed arrays

```bash
SERVERS=("web01" "web02" "db01")      # why: zero-indexed
SERVERS[3]="cache01"                  # why: assign by index
SERVERS+=("lb01")                     # why: append — bash only

echo "${SERVERS[0]}"                  # why: single element; braces are MANDATORY for arrays
echo "${SERVERS[@]}"                  # why: all elements as SEPARATE words — what you almost always want
echo "${SERVERS[*]}"                  # why: all elements as a SINGLE string joined by first char of IFS
echo "${#SERVERS[@]}"                 # why: number of elements
echo "${#SERVERS[1]}"                 # why: LENGTH of element 1 (character count) — easy to confuse
echo "${!SERVERS[@]}"                 # why: all INDICES — needed when the array is sparse
echo "${SERVERS[@]:1:2}"              # why: slice — 2 elements starting at index 1
unset 'SERVERS[2]'                    # why: remove one element (leaves a GAP — array becomes sparse)
```

### `"${arr[@]}"` vs `"${arr[*]}"` — guaranteed interview question

```bash
ARR=("a b" "c d")

for i in "${ARR[@]}"; do echo "[$i]"; done   # [a b]  [c d]   ← 2 iterations, correct
for i in "${ARR[*]}"; do echo "[$i]"; done   # [a b c d]      ← 1 iteration, elements merged
for i in ${ARR[@]};   do echo "[$i]"; done   # [a] [b] [c] [d] ← 4 iterations, UNQUOTED is broken
```

**Always use `"${arr[@]}"` with quotes.**

### Associative arrays (bash 4+)

```bash
declare -A ENVMAP                      # why: MUST declare -A first or bash treats keys as arithmetic → index 0
ENVMAP["dev"]="10.0.1.0/24"
ENVMAP["prod"]="10.0.3.0/24"

echo "${ENVMAP[prod]}"
echo "${!ENVMAP[@]}"                   # why: all KEYS
echo "${ENVMAP[@]}"                    # why: all VALUES

for key in "${!ENVMAP[@]}"; do
  echo "$key → ${ENVMAP[$key]}"        # why: standard key/value iteration pattern
done

[[ -v ENVMAP["staging"] ]] && echo "exists"   # why: key-existence test without creating the key
```

### Loading command output into an array

```bash
mapfile -t PODS < <(kubectl get pods -o name)   # why: mapfile/readarray is the SAFE way; -t strips trailing newlines
readarray -t LINES < /etc/hosts                 # why: readarray is an alias for mapfile

IFS=$'\n' read -r -d '' -a ARR < <(command)     # why: older alternative when mapfile isn't available

PODS=( $(kubectl get pods -o name) )            # why: AVOID — word-splits on spaces, glob-expands *, breaks on weird names
```

### Interview Q&A

**Q: Difference between `${arr[@]}` and `${arr[*]}`?**

> Quoted, `[@]` expands to each element as a separate word; `[*]` joins everything into one string using the first character of `IFS`. In loops and when passing arguments you want `"${arr[@]}"`. `"${arr[*]}"` is only useful when you deliberately want a joined string, like building a comma-separated list by setting `IFS=,`.

**Q: How do you safely read command output into an array?**

> `mapfile -t ARR < <(command)`. The naive `ARR=( $(command) )` performs word splitting and globbing, so filenames with spaces or a literal `*` corrupt the array.

---

## 11. String operations **[MUST KNOW]**

```bash
S="devops-engineer-aakash"

echo "${#S}"                 # why: string length

# ---- substring extraction ----
echo "${S:0:6}"              # why: 6 chars from index 0 → "devops"
echo "${S:7}"                # why: from index 7 to end
echo "${S: -6}"              # why: last 6 chars (SPACE before -6 is required)

# ---- prefix/suffix removal (the ones you actually use) ----
echo "${S#devops-}"          # why: remove SHORTEST match from FRONT
echo "${S##*-}"              # why: remove LONGEST match from FRONT → basename-like → "aakash"
echo "${S%-aakash}"          # why: remove SHORTEST match from END
echo "${S%%-*}"              # why: remove LONGEST match from END → "devops"

# Real usage — pure-bash basename/dirname (no subprocess):
FILE="/var/log/nginx/access.log"
echo "${FILE##*/}"           # why: "access.log" — faster than $(basename "$FILE"), no fork
echo "${FILE%/*}"            # why: "/var/log/nginx" — dirname equivalent
echo "${FILE##*.}"           # why: "log" — file extension
echo "${FILE%.*}"            # why: strip extension

# ---- replacement ----
echo "${S/-/_}"              # why: replace FIRST occurrence
echo "${S//-/_}"             # why: replace ALL occurrences
echo "${S/#devops/DEV}"      # why: replace only if at START
echo "${S/%aakash/RAO}"      # why: replace only if at END
echo "${S//-/}"              # why: delete all occurrences

# ---- case conversion (bash 4+) ----
echo "${S^^}"                # why: ALL UPPERCASE
echo "${S,,}"                # why: all lowercase
echo "${S^}"                 # why: capitalize first char

# ---- comparison & matching ----
[[ "$S" == *"engineer"* ]] && echo "contains"       # why: glob match inside [[ ]] — no quotes on the pattern
[[ "$S" =~ ^devops ]] && echo "regex match"         # why: =~ is bash regex; pattern must be UNQUOTED
[[ -z "$S" ]] && echo "empty"                       # why: -z = zero length
[[ -n "$S" ]] && echo "not empty"                   # why: -n = non-zero length

# ---- split on a delimiter ----
CSV="web01,web02,db01"
IFS=',' read -ra HOSTS <<< "$CSV"                   # why: -r no backslash escapes, -a into array, <<< here-string
printf '%s\n' "${HOSTS[@]}"

# ---- trim whitespace (pure bash) ----
RAW="   padded   "
TRIMMED="${RAW#"${RAW%%[![:space:]]*}"}"            # why: strip leading
TRIMMED="${TRIMMED%"${TRIMMED##*[![:space:]]}"}"    # why: strip trailing
```

**Why parameter expansion over `sed`/`awk` for simple cases:** no subprocess fork. In a loop over 10,000 lines, `${VAR##*/}` vs `$(basename $VAR)` is the difference between instant and ~30 seconds.

### Interview Q&A

**Q: Get the filename from a full path without using `basename`?**

> `"${PATH##*/}"` — remove the longest prefix matching `*/`. For the directory, `"${PATH%/*}"`. I prefer these in loops because they don't fork a subprocess.

**Q: How do you check if a string contains a substring?**

> `[[ "$str" == *"sub"* ]]` for a glob match, or `[[ "$str" =~ sub ]]` for regex. The pattern must not be quoted or it becomes a literal comparison — that's the common mistake.

---

## 12. Getting input from the user **[MUST KNOW]**

```bash
read -p "Enter environment: " ENV          # why: -p prints the prompt on the same line
read -s -p "Password: " PASS; echo         # why: -s = silent (no echo) — ALWAYS use for secrets
read -t 10 -p "Confirm (10s): " ANS        # why: -t = timeout in seconds; prevents a script hanging forever in CI
read -n 1 -p "Continue? [y/n] " YN; echo   # why: -n 1 = read exactly 1 char, no Enter needed
read -r LINE                               # why: -r = raw, do NOT interpret backslashes — use ALWAYS
read -a ARR                                # why: split input into an array on IFS
read -e -p "File: " FILE                   # why: -e enables readline (tab completion) on the prompt
```

### Production-safe input pattern

```bash
read -rp "Delete namespace '$NS'? [yes/NO] " CONFIRM   # why: -r raw, -p prompt, combined
if [[ "${CONFIRM,,}" != "yes" ]]; then                 # why: ,, lowercases so Yes/YES/yes all work
  echo "Aborted."
  exit 1
fi
```

**Important for DevOps:** scripts that run in CI/CD or cron have **no TTY**. `read` will either block forever or get EOF immediately. Guard interactive prompts:

```bash
if [[ -t 0 ]]; then            # why: -t 0 tests if stdin is a terminal
  read -rp "Proceed? " ans
else
  ans="yes"                    # why: non-interactive (CI/cron) → use a safe default or require a --yes flag
fi
```

### Interview Q&A

**Q: Why always use `read -r`?**

> Without `-r`, read interprets backslashes as escape characters, so a Windows path like `C:\new\temp` loses characters. `-r` makes it literal. There's no case where you want the default behaviour — shellcheck flags it as SC2162.

**Q: How do you read a password without it appearing on screen or in history?**

> `read -s -p "Password: " PASS`. But in a DevOps context I'd avoid prompting at all — pull the secret from Vault, AWS Secrets Manager, or the CI secret store, and never pass it as a command-line argument since `ps aux` exposes those.

---

## 13. Arithmetic operations **[MUST KNOW]**

```bash
# ---- the modern way ----
echo $(( 5 + 3 ))            # why: arithmetic expansion — PREFERRED, no subprocess
A=10; B=3
echo $(( A + B ))            # why: $ not needed on variables inside (( ))
echo $(( A / B ))            # why: INTEGER division → 3, not 3.33
echo $(( A % B ))            # why: modulo → 1
echo $(( A ** 2 ))           # why: exponent → 100
(( A++ )); (( A += 5 ))      # why: C-style increment/compound assignment

# ---- arithmetic in conditions ----
if (( A > B )); then echo "greater"; fi    # why: (( )) uses natural operators > < == instead of -gt -lt -eq

# ---- legacy ----
expr 5 + 3                   # why: external binary, forks a process, needs \* for multiply — avoid in new code
let "C = A + B"              # why: bash builtin, works but $(( )) is clearer

# ---- floating point: bash CANNOT do it ----
echo "scale=2; 10/3" | bc              # why: bc with scale for decimal places → 3.33
awk 'BEGIN { printf "%.2f\n", 10/3 }'  # why: awk alternative, often already available
python3 -c "print(10/3)"               # why: when the math gets non-trivial, stop fighting bash
```

### Real DevOps usage

```bash
# Disk usage percentage threshold
USED=$(df -P / | awk 'NR==2 {gsub(/%/,""); print $5}')   # why: -P forces single-line output; gsub strips %
if (( USED > 80 )); then
  echo "ALERT: root filesystem at ${USED}%"
fi

# Memory percentage with bc (needs decimals)
FREE_PCT=$(free -m | awk 'NR==2 {printf "%.1f", $7/$2*100}')   # why: $7 is "available", the number that matters
```

### Interview Q&A

**Q: How do you do floating point math in bash?**

> You don't — bash arithmetic is integer only. I pipe to `bc -l` with a `scale` setting, or use `awk 'BEGIN{printf...}'`. For percentages in monitoring scripts, `awk` is usually already in the pipeline anyway so it's free.

**Q: `$(( ))` vs `expr`?**

> `$(( ))` is a shell builtin — no fork, cleaner syntax, no escaping needed for `*`. `expr` is an external binary from the days before shells had arithmetic. In a loop, `expr` can add measurable overhead per iteration. There's no reason to use `expr` in new scripts.
