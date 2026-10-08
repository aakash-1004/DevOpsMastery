---
title: "Revision — Linux & Bash Scripting"
type: cheatsheet
module: revision
status: ✅ Active Revision
interview-relevance: 🔴 Maximum — Full Stack DevOps Interview Prep
tags:
  - devops
  - revision
  - interview
  - masterreview
  - cheatsheet
order: 2
up: "[[Master-Revision-MOC]]"
prev: "[[Revision-01-DevOps-Fundamentals-and-SDLC]]"
next: "[[Revision-03-Git-and-Version-Control]]"
related:
  - "[[1.2-Shell-Scripting]]"
origin: DevOps-Mastery/Devops master revision.md
---

# Revision — Linux & Bash Scripting

<!-- nav -->
↑ [[Master-Revision-MOC|DevOps Master Revision — Zero to Hero]] · ← [[Revision-01-DevOps-Fundamentals-and-SDLC|Revision — DevOps Fundamentals & SDLC]] · → [[Revision-03-Git-and-Version-Control|Revision — Git & Version Control]]
<!-- /nav -->

## Concept

Linux is the operating system of DevOps. Every server you manage, every container that runs, every CI runner that builds your code — it's Linux. Not knowing Linux deeply is like being a mechanic who doesn't understand engines.

**The DevOps-critical Linux areas:**

1. Process management (ps, top, kill, systemctl)
2. Networking (ss, netstat, iptables, curl, dig)
3. File & text manipulation (grep, awk, sed, find, xargs)
4. Disk & storage (df, du, lsblk, mount, LVM)
5. User & permission management
6. Bash scripting automation

---

## Core Commands

```bash
# Process management
ps aux | grep <process>       # List all processes, filter by name
top -bn1                      # One-shot top output (useful in scripts)
kill -9 <PID>                 # Force kill a process
systemctl status <service>    # Check service status
systemctl restart <service>   # Restart a service
journalctl -u <service> -f    # Follow live logs for a service

# Networking
ss -tulnp                     # Show all listening ports with process names
netstat -tulnp                # Same as ss, older tool
curl -I https://example.com   # HTTP headers only (check if site is up)
dig google.com                # DNS lookup
ip addr show                  # Show all network interfaces and IPs
iptables -L -n -v             # List all firewall rules with verbosity

# File & text manipulation
grep -E "ERROR|WARN" app.log  # Extended regex: find ERRORs or WARNs
grep -r "keyword" /var/log/   # Recursive search in directory
awk '{print $1, $5}' file     # Print columns 1 and 5
awk -F: '{print $1}' /etc/passwd  # Use : as delimiter, print field 1
sed 's/old/new/g' file        # Replace all occurrences of old with new
sed -i 's/old/new/g' file     # In-place replacement (modifies the file)
find /var/log -name "*.log" -mtime +7 -delete  # Delete logs older than 7 days

# Disk & storage
df -h                         # Disk usage in human-readable format
du -sh /var/log/*             # Size of each item in /var/log
lsblk                         # List block devices (disks, partitions)
mount /dev/xvdf /data         # Mount a disk to a directory

# LVM (Logical Volume Management)
pvs / vgs / lvs               # Show physical volumes, volume groups, logical volumes
lvextend -L +5G /dev/vg/lv   # Extend a logical volume by 5GB
resize2fs /dev/vg/lv          # Resize the filesystem after extending LV

# User & permissions
useradd -m -s /bin/bash user  # Create user with home dir and bash shell
usermod -aG sudo user         # Add user to sudo group
passwd user                   # Set password for user
chmod 755 /path               # rwxr-xr-x permissions
chown user:group /path        # Change ownership
```

---

## Bash Scripting — The Production Pattern

```bash
#!/bin/bash
set -euo pipefail

# -e: exit immediately on error
# -u: treat unset variables as errors
# -o pipefail: catch failures in pipes (not just last command)
```

**Logging pattern:**

```bash
log_info()  { echo "[INFO]  $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"; }
log_error() { echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE" >&2; }
```

**Lock file pattern (prevent duplicate cron runs):**

```bash
LOCKFILE="/tmp/script.lock"
cleanup() { rm -f "$LOCKFILE"; }
trap cleanup EXIT                    # Always clean up on exit (even on error)

if [[ -f "$LOCKFILE" ]]; then
  log_error "Already running. Exiting."
  exit 1
fi
touch "$LOCKFILE"
```

**The `|| FAILED=1` pattern (run all checks, fail at end):**

```bash
FAILED=0
check_cpu    || FAILED=1             # Run check; if it fails, set flag — don't exit
check_memory || FAILED=1
check_disk   || FAILED=1

[[ $FAILED -eq 1 ]] && { log_error "One or more checks failed"; exit 1; }
```

**Cron syntax:**

```
* * * * * command
│ │ │ │ └── Day of week (0=Sun, 6=Sat)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)

0 2 * * *  /opt/scripts/backup.sh      # Every day at 2 AM
*/5 * * * * /opt/scripts/health.sh     # Every 5 minutes
```

---

## Senior Interview Q&A

**Q: Production script is running via cron and occasionally hangs. How do you handle it?**

> "Three things. First, lock file — if a previous instance is still running, the new one exits immediately rather than spawning a zombie. Second, timeout wrapper — `timeout 300 ./script.sh` kills it after 5 minutes if it hangs. Third, structured logging to a file with timestamps — so when it does hang, I can see exactly where it stopped.
>
> Beyond that, I'd check what the script is waiting on. Usually it's a network call with no timeout (curl without `--max-time`), or a database query with no timeout set. Those will hang indefinitely. Always set timeouts on any external call inside a script."

---

**Q: Walk me through how you'd debug a server that's responding slowly.**

> "Systematic approach: start with the top-level symptoms and drill down.
>
> First, `top` or `htop` — is CPU maxed? If yes, which process? `top` sorted by CPU shows you immediately.
>
> If CPU is fine, check memory: `free -h`. Is swap being used heavily? Heavy swap means you're OOM-adjacent — the kernel is swapping pages to disk, which destroys performance. Check which process is consuming memory with `ps aux --sort=-%mem | head`.
>
> If CPU and memory are fine, check disk: `iostat -x 1` — are you hitting 100% disk utilization? Are you hitting IOPS limits on EBS?
>
> If that's fine, check network: `ss -tulnp` — are you exhausting connection limits? `netstat -s` shows TCP retransmissions.
>
> Last layer: application-level. Check application logs, error rates, slow query logs in the DB.
>
> In my experience, 80% of slow server issues are either memory pressure causing swap, or a runaway process eating CPU, or a DB query without an index."

---

**Q: What's the difference between `grep`, `awk`, and `sed` and when do you use each?**

> "They solve different problems.
>
> `grep` is a filter — you use it when you want to find lines that match a pattern. `grep ERROR app.log`. It doesn't transform, it selects. Use `-E` for extended regex when you need things like `{1,3}` quantifiers or `|` OR conditions.
>
> `awk` is a column processor. It treats each line as fields separated by a delimiter. Use it when you want to extract or compute on specific columns: `awk '{print $1, $5}'` or `awk -F: '$3 >= 1000 {print $1}' /etc/passwd` to find non-system users. It's also got arithmetic, so you can do `awk '{sum += $2} END {print sum}'` for aggregates.
>
> `sed` is a stream editor for transformations. You use it when you want to modify content in place — replace strings, delete lines, insert text. `sed 's/old/new/g'` for replacement, `sed -i` to modify the file directly.
>
> In a pipeline they complement each other: `grep ERROR app.log | awk '{print $5}' | sort | uniq -c | sort -rn` — filter → extract column → count → rank."

---

**Q: Explain iptables. What are chains and tables?**

> "iptables is Linux's packet filtering framework. It sits in the kernel and makes decisions on every packet.
>
> It has tables for different purposes: `filter` (the default — controls what traffic is allowed), `nat` (network address translation — source/destination NAT), `mangle` (packet marking, QoS), and `raw` (connection tracking bypass).
>
> Within each table are chains — predefined points in the packet path where rules are evaluated: `INPUT` (packets destined for this host), `OUTPUT` (packets originating from this host), `FORWARD` (packets passing through this host as a router), `PREROUTING` (before routing decision), `POSTROUTING` (after routing decision).
>
> Every packet traverses the chains in order. First matching rule wins. If no rule matches, the chain policy applies (ACCEPT or DROP).
>
> In Kubernetes, iptables is what actually implements Service routing. When you create a ClusterIP service, kube-proxy writes iptables rules that NAT the ClusterIP to one of the backend pod IPs. That's why you can't `ping` a ClusterIP — it's a virtual IP that only works via iptables NAT, not ICMP."

<!-- related -->
---
**Related:** [[1.2-Shell-Scripting|Shell Scripting]]
<!-- /related -->
