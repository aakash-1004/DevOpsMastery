---
title: Shell Scripting — Interview Question Bank
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
order: 11
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-10-Common-Gotchas-Cheat-Sheet]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Interview Question Bank

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-10-Common-Gotchas-Cheat-Sheet|Shell Scripting — Common Gotchas Cheat Sheet]]
<!-- /nav -->

## Rapid-fire (one-liner answers expected)

1. **Shebang purpose?** Tells the kernel which interpreter to run the script with.
2. **`#!/bin/bash` vs `#!/usr/bin/env bash`?** Hardcoded path vs PATH lookup; `env` is portable.
3. **`source` vs `./`?** Current shell vs subshell; variables/cd persist only with `source`.
4. **`$?`** Exit code of the last command.
5. **`$$`** PID of the current shell.
6. **`$!`** PID of the last background process.
7. **`$#`** Argument count.
8. **`"$@"` vs `"$*"`?** Separate words vs one joined string.
9. **`[ ]` vs `[[ ]]`?** POSIX test vs bash keyword; `[[ ]]` is safe with empty vars and supports `=~`, globs, `&&`.
10. **`set -e`?** Exit on first failing command.
11. **`set -u`?** Error on unset variables.
12. **`set -o pipefail`?** Pipeline returns the first non-zero exit code.
13. **`set -x`?** Print each command before executing (trace).
14. **Exit 127?** Command not found.
15. **Exit 137?** 128+9 = SIGKILL; in K8s usually OOMKilled.
16. **`trap`?** Run a handler on signals or script exit — used for cleanup.
17. **`/dev/null`?** Discards all input written to it.
18. **`2>&1`?** Redirect stderr to wherever stdout currently points.
19. **`tee`?** Write to a file and stdout simultaneously.
20. **`mktemp`?** Create a secure, uniquely named temp file/dir.
21. **`flock`?** File-based lock — prevents concurrent runs.
22. **`getopts`?** Parse short command-line flags.
23. **`shift`?** Discard `$1` and shift the remaining args left.
24. **`IFS`?** Internal Field Separator — controls word splitting.
25. **`local`?** Scope a variable to a function.
26. **`readonly`?** Make a variable immutable.
27. **`shellcheck`?** Static analysis linter for shell scripts.
28. **`$(( ))` vs `expr`?** Builtin arithmetic, no fork, vs external binary.
29. **Floating point in bash?** Not supported — use `bc` or `awk`.
30. **Difference between `break` and `continue`?** Exit the loop vs skip to the next iteration.

## Conceptual

**Q: Walk me through what happens when you run `./script.sh`.**

> The shell forks, the kernel's `execve` reads the first two bytes, sees `#!`, and re-executes with the named interpreter passing the script path as an argument. The new bash process reads the file, parses each command, performs expansions — brace, tilde, parameter, command substitution, arithmetic, then word splitting and globbing — and forks/execs each external command, wiring up file descriptors per the redirections. The script's exit code is the exit code of its last command unless there's an explicit `exit`.

**Q: When would you NOT use bash?**

> When the script needs real data structures, JSON/YAML manipulation beyond a simple `jq` call, concurrency with shared state, proper exception handling, or unit tests. Also once it crosses roughly 300 lines, because bash's error handling and scoping don't scale. At that point I move to Python. And for configuration management specifically I'd reach for Ansible or Terraform rather than writing imperative bash, because they handle idempotency and state for me.

**Q: How do you make a script idempotent, and why does it matter?**

> Idempotent means running it twice gives the same result as running it once. I use `mkdir -p`, `ln -sfn`, check-before-append with `grep -q || echo >>`, `systemctl enable --now`, and `aws s3 sync` rather than `cp`. It matters because automation gets re-run — a retried CI job, a cron job that overlaps, a failed partial run. If the script isn't idempotent, a retry makes things worse instead of converging.

**Q: Your script needs a secret. How do you handle it?**

> Never on the command line, because `ps aux` exposes arguments to every user on the box. Never in the script or in a logged variable. I pull it at runtime from AWS Secrets Manager, Vault, or the CI secret store, keep it in a variable with `set +x` around that block so tracing doesn't leak it, and if it must hit disk it goes to a `chmod 600` file that a `trap cleanup EXIT` removes. For Docker, `--env-file` not `-e`. And if a secret ever does get committed or logged, rotation comes first — removing it from history afterwards is not enough on its own.

**Q: How would you process a 50 GB log file in bash?**

> Stream it, never load it. `awk` or `grep` reading line by line keeps memory flat. I'd avoid anything that buffers the whole file — no `$(cat file)`, no arrays of all lines. If it needs multiple passes I'd use `awk` to do everything in one pass, and if it's genuinely CPU-bound, `parallel` or `xargs -P` splitting by chunk. Also `LC_ALL=C` in front of grep/sort, which can be several times faster by skipping UTF-8 collation.

**Q: How do you ensure quality on shell scripts in a team?**

> shellcheck in CI as a required check, a shared template with `set -euo pipefail` and a standard logging/trap block, and a lib directory of sourced common functions so error handling isn't reinvented per script. Code review focused on quoting and destructive operations specifically. And a mandatory dry-run flag on anything that deletes or modifies infrastructure.

## Scenario-based

**Q: A cron job has been "running" for 3 months but the backups don't exist. How do you investigate?**

> First, did cron fire it at all — `grep CRON /var/log/syslog` or `journalctl -u cron` for the schedule. If it fired, the script failed silently, almost certainly because the crontab entry discards output. I'd look for `>/dev/null 2>&1` and replace it with a real log path, then run the script manually as the cron user with `sudo -u backup bash -x /path/script.sh` to see where it diverges. The usual causes are PATH (the `aws` binary not found), missing credentials because `~/.aws/credentials` belongs to a different user, or an unescaped `%` in a `date` format breaking the command. Longer term, the actual fix is that the job needs to report success to monitoring — a dead-man's-switch check that alerts when a heartbeat _stops_ arriving, because "silently never ran" is invisible to a normal alert.

**Q: A deployment script left a Kubernetes namespace half-created when someone Ctrl+C'd it. How do you prevent that?**

> `trap cleanup EXIT INT TERM` with a cleanup function that either rolls back or at minimum records the partial state. But the better answer is to stop writing imperative deploy scripts — declare the desired state in manifests and let a reconciler converge it. That's exactly what ArgoCD does in a GitOps setup: the state lives in Git, and a Ctrl+C on a client machine can't leave the cluster in a half-applied state because the controller keeps reconciling toward the committed state.

**Q: You need to check 200 servers for a specific package version. How?**

> I wouldn't write a sequential bash loop — 200 serial SSH connections at 2 seconds each is seven minutes. `xargs -P 20` or GNU `parallel` to run 20 at a time, with `ssh -n -o ConnectTimeout=5 -o BatchMode=yes` so a dead host fails fast instead of hanging and prompting for a password. Output to a structured format so I can sort the results. In practice though, if there are 200 servers there should already be a config management tool — Ansible's `ad-hoc` mode does this in one command with proper parallelism and output handling, and AWS Systems Manager Run Command does it with no SSH at all.

**Q: Explain this: `find / -name "*.log" | xargs rm` and what's wrong with it.**

> Three problems. It breaks on filenames containing spaces or newlines because xargs splits on whitespace — a file called `my log.log` becomes two arguments and you delete the wrong things. It runs from `/`, so it'll traverse `/proc`, `/sys`, and any network mounts. And if find returns nothing, some xargs implementations still run `rm` once with no arguments. The safe version is `find /var/log -name "*.log" -type f -delete`, or if you need xargs, `find … -print0 | xargs -0 -r rm --`.
