---
title: Shell Scripting — Production Script Template
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
order: 9
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-08-Projects]]"
next: "[[Shell-10-Common-Gotchas-Cheat-Sheet]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Production Script Template

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-08-Projects|Shell Scripting — Projects (RAM monitor, disk alert, log archive, user management)]] · → [[Shell-10-Common-Gotchas-Cheat-Sheet|Shell Scripting — Common Gotchas Cheat Sheet]]
<!-- /nav -->

```bash
#!/usr/bin/env bash
#
# <name>.sh — <one-line purpose>
# Usage : <invocation>
# Exit  : 0 ok | 1 <reason> | 2 <reason>
#
set -euo pipefail
IFS=$'\n\t'

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_NAME="${BASH_SOURCE[0]##*/}"
readonly SCRIPT_DIR SCRIPT_NAME

# --- config (env-overridable with sane defaults) ---
LOG_FILE="${LOG_FILE:-/var/log/${SCRIPT_NAME%.sh}.log}"
DRY_RUN="${DRY_RUN:-false}"
ENVIRONMENT="${ENVIRONMENT:?ENVIRONMENT must be set}"

# --- logging ---
log()  { printf '%s [%-5s] %s\n' "$(date '+%F %T')" "$1" "${*:2}" | tee -a "$LOG_FILE" >&2; }
info() { log INFO  "$@"; }
warn() { log WARN  "$@"; }
err()  { log ERROR "$@"; }
die()  { err "$@"; exit 1; }

# --- cleanup ---
TMPDIR="$(mktemp -d)"
cleanup() {
  local rc=$?
  rm -rf "$TMPDIR"
  (( rc == 0 )) && info "Completed in ${SECONDS}s" || err "Failed (rc=$rc) after ${SECONDS}s"
  exit "$rc"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM
trap 'err "Error at ${BASH_SOURCE[0]}:${LINENO}: $BASH_COMMAND"' ERR

# --- preflight ---
for cmd in kubectl aws jq; do
  command -v "$cmd" >/dev/null || die "Required command not found: $cmd"
done

# --- main ---
main() {
  info "Starting on $(hostname -s) for env=$ENVIRONMENT"
  # ... work ...
}

main "$@"
```

Memorise the shape of this. Being able to produce it in a live coding round is a strong differentiator.
