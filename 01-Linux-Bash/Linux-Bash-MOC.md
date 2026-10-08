---
title: "Linux & Bash — Map of Content"
type: moc
module: linux
tags:
  - moc
  - linux
up: "[[Home]]"
---

# Linux & Bash — Map of Content

<!-- nav -->
↑ [[Home|Home]]
<!-- /nav -->

Linux internals, networking and performance troubleshooting; bash scripting from strict mode to a full reference; cron and log parsing.

**Related modules:** [[Python-DevOps-MOC|Python for DevOps]] · [[Deployment-MOC|Deployment (Nginx, PM2, Gunicorn, systemd)]] · [[Monitoring-MOC|Monitoring]]

## Linux core
- [[1.1-Linux-Core-Concepts|Linux — Core Concepts]] — *concept* · interview: 🔴 High — tested in every DevOps/SRE interview
  - Covers: The Kernel; Filesystem Hierarchy; Processes; File Permissions; Interview-Ready Spoken Answers
- [[1.3-Linux-Networking-Commands|Linux — Networking Commands]] — *concept* · interview: 🔴 High — troubleshooting network issues is a daily DevOps task
  - Covers: curl — HTTP Requests from Terminal; ss / netstat — Network Connections; iptables — Linux Firewall; dig / nslookup — DNS Lookup; Other Essential Commands; Interview-Ready Spoken Answers
- [[1.4-Linux-Performance-Troubleshooting|Linux — Performance & Troubleshooting]] — *concept* · interview: 🔴 High — troubleshooting scenario questions are standard in DevOps interviews
  - Covers: top / htop — Live System Monitor; vmstat — System Performance Snapshot; iotop — Disk I/O Per Process; lsof — List Open Files; strace — System Call Tracer; dmesg — Kernel Messages; free — Memory Usage

## Bash & scripting
- [[1.2-Shell-Scripting|Shell Scripting]] — *concept* · interview: 🔴 High — scripting questions in every DevOps interview
  - Covers: Variables; Special Variables; Conditionals; Loops; Functions; Safety Flags — Always Use These; Real-World Scripts
- [[1.5-Bash-Strict-Mode|Bash Strict Mode]] — *concept* · interview: 🔴 High — asked in almost every DevOps interview
  - Covers: What Is It?; Why It Matters; Conditionals; Interview — Ready to Speak
- [[1.6-Cron-Jobs|Cron Jobs]] — *concept* · interview: 🟡 Medium — comes up in ops/automation questions
  - Covers: Cron Syntax; Setting Up a Cron Job; Critical Gotchas; Verify Cron Is Running; Interview — Ready to Speak
- [[1.7-Log-Parsing|Log Parsing — grep, awk, sed]] — *concept* · interview: 🔴 High — Day 1 task on any DevOps/SRE job
  - Covers: grep — Find Lines; awk — Extract Fields & Compute; sed — Find, Replace, Delete; Real Combo — CPU Usage Extraction; Interview — Ready to Speak

## Labs
- [[Lab-Server-Health-Check-Script|Lab — Server Health Check Script]] — *lab*
  - Covers: What It Does; Key Pattern — Run All Checks, Collect Failures; Bug Found During Lab; Test Cases Run
- [[Lab-Postfix-mailx-Gmail-Relay|Postfix + mailx Setup (Ubuntu, Gmail Relay)]] — *lab*
  - Covers: Install; Verify the service; Core config (`/etc/postfix/main.cf`); Gmail relay with SASL auth; Aliases and apply; Test; Troubleshooting
- [[Lab-Logging-Trap-Lock-Files|Logging, Trap & Lock Files]] — *lab* · interview: 🔴 High — production scripting pattern, shows maturity
  - Covers: Logging Functions; Trap & Cleanup; Lock File Pattern; Interview — Ready to Speak

## Notes
- [[Shell-Scripting-Reference-MOC|Shell Scripting — Complete DevOps Reference]] — *hub* · 11 notes

[[Home]] · [[Interview-Prep-MOC]]
