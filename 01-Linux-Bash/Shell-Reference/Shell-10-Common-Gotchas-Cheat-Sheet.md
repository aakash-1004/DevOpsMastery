---
title: Shell Scripting — Common Gotchas Cheat Sheet
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
order: 10
up: "[[Shell-Scripting-Reference-MOC]]"
prev: "[[Shell-09-Production-Script-Template]]"
next: "[[Shell-11-Interview-Question-Bank]]"
origin: DevOps-Mastery/01-Linux-Bash/0.0- Shell Scripting-all.md
---

# Shell Scripting — Common Gotchas Cheat Sheet

<!-- nav -->
↑ [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] · ← [[Shell-09-Production-Script-Template|Shell Scripting — Production Script Template]] · → [[Shell-11-Interview-Question-Bank|Shell Scripting — Interview Question Bank]]
<!-- /nav -->

|Gotcha|Wrong|Right|
|---|---|---|
|Unquoted variable|`rm $FILE`|`rm -- "$FILE"`|
|Spaces around `=`|`VAR = "x"`|`VAR="x"`|
|Reading a file|`for l in $(cat f)`|`while IFS= read -r l; do … done < f`|
|Counter in a pipe|`cat f \| while read …`|`while read … done < f`|
|Find + loop|`for f in $(find …)`|`find … -print0 \| while IFS= read -r -d ''`|
|Array into command|`${arr[*]}`|`"${arr[@]}"`|
|Forwarding args|`"$*"`|`"$@"`|
|Redirect order|`2>&1 > file`|`> file 2>&1`|
|Group append|`usermod -G sudo u`|`usermod -aG sudo u`|
|Check `$?` late|`cmd; echo x; [[ $? …]]`|`if ! cmd; then …`|
|Pipeline exit code|relying on `$?`|`set -o pipefail` + `${PIPESTATUS[@]}`|
|`%` in crontab|`date +%F`|`date +\%F`|
|Secrets on CLI|`mysql -p"$PASS"`|`--defaults-extra-file` / `--env-file`|
|Monitoring memory|`free` column|`available` column (`$7`)|
|Disk full mystery|only `df -h`|also `df -i` (inodes)|
|ssh in a loop|`ssh "$h" cmd`|`ssh -n "$h" cmd`|
|Infinite loop|no sleep|`sleep N` inside|
|Temp files|`/tmp/mydir`|`$(mktemp -d)` + `trap cleanup EXIT`|
|CRLF line endings|editing on Windows|`dos2unix script.sh`|
|`#!/bin/sh` on Ubuntu|assumes bash|it's dash — use `#!/usr/bin/env bash`|
