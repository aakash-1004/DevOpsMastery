---
title: "Postfix + mailx Setup (Ubuntu, Gmail Relay)"
type: lab
module: linux
tags:
  - linux
  - postfix
  - mailx
  - smtp
  - gmail
  - lab
created: "2026-10-02"
up: "[[Linux-Bash-MOC]]"
prev: "[[Lab-Server-Health-Check-Script]]"
next: "[[Lab-Logging-Trap-Lock-Files]]"
origin: "DevOps-Mastery/01-Linux-Bash/Labs/Postfix + mailx Setup (Ubuntu, Gmail Relay).md"
---

# Postfix + mailx Setup (Ubuntu, Gmail Relay)

<!-- nav -->
↑ [[Linux-Bash-MOC|Linux & Bash]] · ← [[Lab-Server-Health-Check-Script|Lab — Server Health Check Script]] · → [[Lab-Logging-Trap-Lock-Files|Logging, Trap & Lock Files]]
<!-- /nav -->

Host: `devops-labs` | Date: 2026-10-02 | Mode: Internet Site, relaying via Gmail SMTP

---

## 1. Install

```bash
sudo apt install postfix          # choose "Internet Site" when prompted
sudo apt install bsd-mailx        # `mailx` is a virtual package, install bsd-mailx directly
sudo apt install libsasl2-modules # required for SMTP auth, otherwise auth fails silently
```

- `mailutils` (GNU) is the alternative to `bsd-mailx`. It pulls in more and has different flags.
- `Package 'mailx' has no installation candidate` is expected on Ubuntu. Install one of the providers.

## 2. Verify the service

```bash
systemctl status postfix
```

`Active: active (exited)` with `ExecStart=/bin/true` is **normal** on Ubuntu/Debian. `postfix.service` is only a wrapper. The real daemon:

```bash
systemctl status postfix@-
sudo postfix status
```

## 3. Core config (`/etc/postfix/main.cf`)

Use `postconf -e` to avoid syntax mistakes. Leave `master.cf` at defaults.

```bash
sudo postconf -e 'inet_interfaces = loopback-only'
sudo postconf -e 'inet_protocols = ipv4'
sudo postconf -e 'myhostname = devops-labs.local'
sudo postconf -e 'mydestination = $myhostname, localhost.$mydomain, localhost'
```

|Setting|Why|
|---|---|
|`inet_interfaces = loopback-only`|Internet Site defaults to `all`, which opens port 25 on every interface. Send-only box, so close it.|
|`inet_protocols = ipv4`|Avoids delays and timeouts when IPv6 isn't routed.|
|`myhostname`|Use a proper FQDN if available. Keep it consistent with `/etc/mailname`.|
|`mydestination`|Domains accepted for local delivery. Keep minimal.|

## 4. Gmail relay with SASL auth

```bash
sudo postconf -e 'relayhost = [smtp.gmail.com]:587'
sudo postconf -e 'smtp_sasl_auth_enable = yes'
sudo postconf -e 'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd'
sudo postconf -e 'smtp_sasl_security_options = noanonymous'
sudo postconf -e 'smtp_tls_security_level = encrypt'
sudo postconf -e 'smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt'
```

Credentials file:

```bash
sudo bash -c 'echo "[smtp.gmail.com]:587 youraddress@gmail.com:your-app-password" > /etc/postfix/sasl_passwd'
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd.db
```

Notes:

- Square brackets around the host stop Postfix doing an MX lookup.
- Gmail needs 2FA enabled and an **app password**, not the account password.
- Gmail rewrites the From address to the authenticated account.
- Never commit or share `sasl_passwd`.

## 5. Aliases and apply

```bash
echo "root: youraddress@gmail.com" | sudo tee -a /etc/aliases
sudo newaliases
sudo systemctl reload postfix
```

## 6. Test

```bash
postconf -n                                   # review effective config
echo "relay test" | mail -s "relay test" youraddress@gmail.com
sudo tail -f /var/log/mail.log                # or: sudo journalctl -u postfix@- -f
```

Success looks like: `status=sent (250 2.0.0 OK)`

## 7. Troubleshooting

|Symptom|Cause / Fix|
|---|---|
|`SASL authentication failed`|Wrong app password, or `libsasl2-modules` missing. Re-run `postmap` after editing `sasl_passwd`.|
|Connection timeout to port 587|Outbound 587 blocked by network/ISP/firewall. Test with `nc -zv smtp.gmail.com 587`.|
|Mail stuck in queue|`mailq` to inspect, `sudo postqueue -f` to flush, `sudo postsuper -d ALL` to purge.|
|Mail lands in spam|Expected without SPF/DKIM/rDNS. Relaying through Gmail/SES avoids most of this.|
|Config change not applied|`sudo systemctl reload postfix`|

## 8. Useful commands

```bash
postconf -n                    # non-default config
postconf -d <param>            # default value of a param
mailq                          # show queue
sudo postqueue -f              # flush queue
sudo postsuper -d <queue_id>   # delete one queued message
sudo postfix check             # config sanity check
```

## 9. JS7 note

- JOC can send notifications via SMTP directly, with no local Postfix needed.
- Local Postfix + `mail` only matters when job scripts call `mail` themselves.
