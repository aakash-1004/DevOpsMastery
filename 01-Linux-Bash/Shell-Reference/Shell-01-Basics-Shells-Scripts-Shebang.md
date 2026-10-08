---
title: "Shell Scripting — Basics: Shells, First Script, Shebang, Running"
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
order: 1
up: "[[Shell-Scripting-Reference-MOC]]"
next: "[[Shell-02-Variables-Arrays-Strings-Input-Arithmetic]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Basics: Shells, First Script, Shebang, Running

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · → [[Shell-02-Variables-Arrays-Strings-Input-Arithmetic|Shell Scripting — Variables, Arrays, Strings, Input, Arithmetic]]
<!-- /nav -->

## 0. How shell scripting is actually used in a DevOps job

Be clear on this before the syntax, because interviewers ask "where do you actually use bash?"

|Real usage|Example|
|---|---|
|Container entrypoints|`entrypoint.sh` that waits for DB, runs migrations, then `exec`s the app|
|CI/CD pipeline steps|GitHub Actions `run:` blocks, Jenkins `sh` steps|
|Health / readiness probes|`exec` probe in a K8s Pod spec|
|Node bootstrap|EC2 user-data, AMI baking, agent install|
|Log rotation / archival / cleanup|cron jobs on app servers|
|Monitoring + alerting glue|disk/RAM thresholds → email/Slack webhook|
|Backup jobs|`etcd` snapshot, DB dump, upload to S3|
|Incident triage one-liners|parsing logs under pressure on a prod box|

**Spoken answer for "do you use shell scripting?"**

> Daily. Entrypoint scripts in containers, pipeline steps in CI, cron-driven housekeeping on servers, and ad-hoc log parsing during incidents. For anything that needs real data structures or API handling I move to Python, but for process orchestration and gluing CLI tools together, bash is the right tool.

---

## 1. What is a Linux Shell? **[MUST KNOW]**

A shell is the **command interpreter** between you and the kernel. It parses what you type, expands it, forks processes, wires up file descriptors, and hands syscalls to the kernel.

Two faces of the same program:

- **Interactive shell** — you type, it responds (prompt, history, job control, tab completion).
- **Non-interactive shell** — runs a script, no prompt, no history.

```
You → Shell (parse, expand, fork/exec) → Kernel → Hardware
```

The shell is just a userspace program. It is not part of the kernel.

### Interview Q&A

**Q: Difference between a shell and a terminal?**

> Terminal (or terminal emulator) is the I/O device — it renders text and captures keystrokes. The shell is the program running _inside_ it that interprets commands. SSH gives you a shell without a local terminal emulator; `gnome-terminal` gives you a terminal that then launches a shell.

**Q: What's the difference between a login shell and a non-login shell?**

> A login shell is the first shell of a session — SSH login, console login, or `bash -l`. It reads `/etc/profile` and `~/.bash_profile` / `~/.profile`. A non-login interactive shell (new terminal tab) reads `~/.bashrc`. This matters constantly in practice: environment variables exported in `~/.bash_profile` won't exist in a non-login shell, which is why cron jobs and CI steps "lose" your PATH.

---

## 2. Types of Shell **[MUST KNOW]**

|Shell|Binary|Notes|
|---|---|---|
|**sh**|`/bin/sh`|POSIX shell. On Ubuntu/Debian it's a symlink to **dash**, not bash. On RHEL it's bash in POSIX mode.|
|**bash**|`/bin/bash`|Bourne Again Shell. The default on most Linux distros. Arrays, `[[ ]]`, process substitution.|
|**zsh**|`/bin/zsh`|macOS default since Catalina. Better interactive UX.|
|**ksh**|`/bin/ksh`|Korn shell. Common on AIX/Solaris legacy.|
|**csh/tcsh**|`/bin/csh`|C-like syntax. Avoid for scripting.|
|**fish**|`/usr/bin/fish`|Friendly interactive shell, **not POSIX compatible**.|

**The one that bites you in production:** on Debian/Ubuntu, `#!/bin/sh` means **dash**, which has no arrays, no `[[ ]]`, no `source` (only `.`), no `local` guarantees. A script that works when you run `bash script.sh` can fail in a Dockerfile `RUN` step or a cron entry that uses `sh`.

### Interview Q&A

**Q: Why does my script work with `bash script.sh` but fail with `sh script.sh`?**

> Because on Debian-based systems `/bin/sh` is dash, a minimal POSIX shell. Bash-specific features — arrays, `[[ ]]`, `+=`, `source`, `function` keyword, brace expansion — aren't available. The fix is either a correct shebang (`#!/usr/bin/env bash`) and executing the script directly, or writing POSIX-compliant code if the script must be portable. Alpine containers hit this hardest because their default shell is BusyBox `ash`.

**Q: Why do Alpine-based container entrypoints break?**

> Alpine ships BusyBox `ash`, and bash isn't installed. Either `apk add --no-cache bash` or write the entrypoint POSIX-compliant. In a slim image I'd prefer POSIX-compliant over adding a package.

---

## 3. How to check shell type **[MUST KNOW]**

```bash
echo $SHELL            # why: user's DEFAULT login shell from /etc/passwd — NOT necessarily the one you're in right now
echo $0                # why: the shell actually executing right now (most accurate for interactive use)
ps -p $$               # why: $$ is the current shell PID; ps shows the real process name — most reliable method
readlink -f /bin/sh    # why: proves whether /bin/sh is dash or bash on this box
cat /etc/shells        # why: lists all valid login shells installed on the system
echo $BASH_VERSION     # why: empty if you're not in bash — a quick bash-vs-not test
grep "^$USER:" /etc/passwd   # why: shows the configured login shell for your user
chsh -s /bin/bash      # why: change your default login shell permanently
```

**Trap:** `$SHELL` is an environment variable set at login. If you then run `zsh`, `$SHELL` still says bash. Use `ps -p $$` when it matters.

### Interview Q&A

**Q: `echo $SHELL` says bash but your script behaves like dash. Why?**

> `$SHELL` reports the configured login shell, not the interpreter running the script. The script's interpreter is decided by its shebang, or by whatever binary was used to invoke it. I'd check `ps -p $$` inside the script or add `echo $BASH_VERSION` to confirm.

---

## 4. What is shell scripting? **[MUST KNOW]**

A shell script is a text file containing shell commands, executed sequentially by an interpreter. It adds variables, conditionals, loops, and functions to what you'd otherwise type by hand.

**When bash is the right tool:**

- Orchestrating other CLI tools (`kubectl`, `aws`, `docker`, `jq`, `curl`)
- Process/file plumbing, exit-code chaining, redirection
- Script < ~200 lines, no complex data structures

**When to switch to Python/Go:**

- JSON/YAML manipulation beyond simple `jq` calls
- Real error handling, retries with state, concurrency
- Anything that needs tests, modules, or will outlive the quarter
- Script is creeping past 300 lines

Interviewers like this answer because it shows judgment, not just syntax.

---

## 5. Creating your first shell script **[MUST KNOW]**

```bash
vim deploy.sh          # why: .sh extension is convention only — the kernel ignores it, the shebang decides the interpreter
```

```bash
#!/usr/bin/env bash
# deploy.sh — minimal skeleton

set -euo pipefail      # why: fail fast; explained in detail in section 18 — put this in every script you write

echo "Hello, $(whoami)"
```

```bash
chmod +x deploy.sh     # why: add execute permission so the kernel will run it via its shebang
./deploy.sh            # why: ./ is required because . is not in PATH on Linux (deliberate security decision)
```

**Naming conventions that matter in a repo:**

- `lowercase-with-hyphens.sh` or `snake_case.sh` — be consistent
- `entrypoint.sh`, `healthcheck.sh`, `bootstrap.sh` — recognised names in container/infra repos
- Scripts meant to be on `$PATH` as commands drop the `.sh` extension (`/usr/local/bin/backup-etcd`)

---

## 6. The shebang line **[MUST KNOW]**

```bash
#!/bin/bash              # absolute path, fails if bash lives elsewhere (e.g. NixOS, some BSD)
#!/usr/bin/env bash      # PREFERRED: resolves bash via PATH, portable across distros and containers
#!/bin/sh                # POSIX mode — dash on Debian/Ubuntu
#!/usr/bin/env python3   # shebangs are not bash-only
#!/bin/bash -e           # works but fragile; prefer `set -e` on line 2 (only ONE arg is portable after the interpreter)
```

Mechanics: the kernel reads the first two bytes `#!`, takes the rest of the line as the interpreter path, and executes `interpreter /path/to/script`. If the shebang is missing, the _calling shell_ decides — usually by running it with `/bin/sh`, which is where portability bugs come from.

**Gotchas that actually happen:**

- A script edited on Windows gets `\r\n` line endings. The shebang becomes `#!/bin/bash\r`, and the kernel tries to execute a binary literally named `bash\r` → `bad interpreter: No such file or directory`. Fix with `dos2unix script.sh` or `sed -i 's/\r$//' script.sh`.
- The shebang must be the **very first line**. A blank line or comment above it breaks it.

### Interview Q&A

**Q: `#!/bin/bash` vs `#!/usr/bin/env bash`?**

> `#!/bin/bash` hardcodes a path. `#!/usr/bin/env bash` looks bash up in `$PATH`, so it works across distros, on macOS where Homebrew bash is `/opt/homebrew/bin/bash`, and in containers with non-standard layouts. I default to `env bash`. The one exception is high-security contexts where a predictable PATH matters more than portability.

**Q: What if a script has no shebang?**

> Executing it directly makes the kernel fail, and the calling shell usually falls back to running it with `/bin/sh`. On Ubuntu that's dash, so any bash-specific syntax breaks. Never ship a script without a shebang.

**Q: "bad interpreter" error — what's your first check?**

> CRLF line endings. `file script.sh` or `cat -A script.sh` will show `^M` at line ends. `dos2unix` fixes it.

---

## 7. How to run a shell script **[MUST KNOW]**

```bash
./script.sh            # why: executes via shebang in a NEW child process (subshell). Needs +x.
bash script.sh         # why: explicitly invoke bash; shebang is IGNORED. No +x needed.
sh script.sh           # why: runs under POSIX sh — this is where "works on my machine" bugs appear
bash -x script.sh      # why: trace mode without editing the file
source script.sh       # why: runs in the CURRENT shell — variables and functions persist after it exits
. script.sh            # why: POSIX equivalent of source
/full/path/script.sh   # why: absolute path, required in cron and systemd units
```

### `source` vs `./` — the classic interview question

||`./script.sh`|`source script.sh`|
|---|---|---|
|Process|New subshell (fork)|Current shell|
|Needs `+x`|Yes|No|
|Shebang honored|Yes|No — runs in current interpreter|
|Variables persist after exit|**No**|**Yes**|
|`cd` affects your shell|**No**|**Yes**|
|`exit` inside it|Kills the script|**Kills your shell/session**|

```bash
# Proof:
echo 'MYVAR=hello' > t.sh
bash t.sh; echo "$MYVAR"     # why: prints empty — child shell's variable died with the child
source t.sh; echo "$MYVAR"   # why: prints "hello" — ran in current shell
```

**Where this matters in real work:** a script that does `cd /opt/app` or exports credentials must be `source`d, not executed — otherwise it changes the directory of a child process that immediately exits. This is why `source venv/bin/activate` and `. ~/.nvm/nvm.sh` exist.

### Interview Q&A

**Q: Your script does `cd /var/log` but your shell stays put after it runs. Why?**

> The script ran in a subshell. `cd` changed the child process's working directory, and the child exited. Use `source script.sh` to run it in the current shell, or have the script output the path and use `cd "$(./script.sh)"`.

---

## 8. Comments **[MUST KNOW]**

```bash
# single line comment

: '
multi-line comment using the null command
everything here is ignored
'

<<'COMMENT'
Alternative multi-line block using an unexecuted here-doc.
COMMENT
```

`#` after the shebang is the only comment style bash truly has. The others are tricks — `:` is the null builtin that ignores its argument.

**Comment discipline for production scripts** — this is what reviewers look for:

```bash
#!/usr/bin/env bash
#
# backup-etcd.sh
# Purpose : Snapshot etcd and upload to S3
# Usage   : ./backup-etcd.sh <s3-bucket>
# Cron    : 0 2 * * * /usr/local/bin/backup-etcd.sh prod-backups
# Author  : platform-team
# Exit    : 0 ok | 1 snapshot failed | 2 upload failed
#
```

Don't comment _what_ the code does (`# loop over files`). Comment _why_ (`# why: ECR login token expires in 12h, so re-auth each run`).
