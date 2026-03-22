---
layout: default
title: "How to Use chkrootkit and rkhunter"
description: "Run chkrootkit and rkhunter to scan Linux systems for rootkits, backdoors, and suspicious files, interpret results, and eliminate false positives"
date: 2026-03-22
author: theluckystrike
permalink: /chkrootkit-rkhunter-rootkit-detection-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Use chkrootkit and rkhunter

chkrootkit and rkhunter are command-line rootkit scanners for Linux. They check for known rootkit signatures, suspicious file permissions, hidden processes, and system binary modifications. Neither tool is perfect — rootkits can evade them — but they catch common infections and are worth running regularly as part of a defense-in-depth strategy.

## Installing Both Tools

```bash
# Debian/Ubuntu
sudo apt install chkrootkit rkhunter

# RHEL/CentOS/Fedora
sudo yum install chkrootkit rkhunter

# Verify versions
chkrootkit -V
rkhunter --version
```

Install on a clean system, then run a baseline scan. If you install after a suspected compromise, the scanners themselves may be compromised — boot from external media instead.

## Running chkrootkit

```bash
# Basic scan
sudo chkrootkit

# Quiet mode — only show infected/suspect results
sudo chkrootkit -q

# Test a specific rootkit
sudo chkrootkit lrk5

# Run against a different root (useful for forensic analysis of a mounted disk)
sudo chkrootkit -r /mnt/compromised-disk
```

### Interpreting chkrootkit Output

```
ROOTDIR is `/'
Checking `amd'...                          not found
Checking `basename'...                     not infected
Checking `biff'...                         not found
Checking `chfn'...                         not infected
...
Checking `bindshell'...                    not infected
Checking `lkm'...                          chkproc: nothing detected
Checking `rexedcs'...                      not found
Checking `sniffer'...                      eth0: not promisc and no packet sniffer sockets
Checking `w55808'...                       not infected
Checking `wted'...                         chkwtmp: nothing deleted
Checking `scalper'...                      not infected
Checking `slapper'...                      not infected
Checking `z2'...                           chklastlog: nothing deleted
```

Key status values:
- **not found** — the binary checked does not exist on your system (normal for server installs)
- **not infected** — binary matches expected patterns
- **INFECTED** — binary matches a rootkit signature (investigate immediately)
- **suspicious** — unusual but not definitively malicious

### Common chkrootkit False Positives

```bash
# chkrootkit sometimes flags these as suspicious on clean systems:

# /tmp/.ICE-unix — X11 socket, normal on desktop systems
# bindshell — a port open on certain services can trigger this
# eth0 in promiscuous mode — happens with packet capture tools, VMs, Docker

# Check what is actually listening
sudo ss -tlnp
sudo ip link show | grep PROMISC
```

## Running rkhunter

rkhunter has a more extensive check list and a properties database that tracks file hashes for system binaries.

```bash
# Update the rkhunter data files first
sudo rkhunter --update

# Run a full scan
sudo rkhunter --check

# Non-interactive (for cron/scripts)
sudo rkhunter --check --skip-keypress

# Show only warnings and errors
sudo rkhunter --check --skip-keypress 2>&1 | grep -E "Warning|Infected|Found"

# Check a specific binary
sudo rkhunter --check-modes /usr/sbin/sshd
```

### Initial Properties Baseline

The first time you run rkhunter after a clean install, store the file properties as the baseline:

```bash
# Build the initial database of known-good binary hashes
sudo rkhunter --propupd

# After any package update, update the database:
sudo apt upgrade && sudo rkhunter --propupd
```

If you forget to run `--propupd` after an update, rkhunter will flag all updated binaries as suspicious.

### rkhunter Configuration

```bash
# /etc/rkhunter.conf — key settings

# Email alerts
MAIL-ON-WARNING=admin@yourdomain.com
MAIL_CMD=sendmail

# Whitelist false positives
SCRIPTWHITELIST=/usr/sbin/adduser
SCRIPTWHITELIST=/usr/bin/ldd

# Allow hidden directories that are legitimate
ALLOWHIDDENDIR=/dev/.udev
ALLOWHIDDENDIR=/dev/.static
ALLOWHIDDENDIR=/dev/.initramfs

# Allow SSH root login if you have a reason for it
ALLOW_SSH_ROOT_USER=no

# Update mirrors (for --update)
UPDATE_MIRRORS=1
MIRRORS_MODE=0

# Log file
LOGFILE=/var/log/rkhunter.log
```

```bash
# Validate your config
sudo rkhunter --config-check
```

### Interpreting rkhunter Warnings

```
[10:30:45]   Checking for suspicious (large) shared memory segments  [ None found ]
[10:30:47]   Checking for hidden files and directories               [ Warning ]
[10:30:47] Warning: Hidden directory found: /dev/.udev

[10:30:49]   Checking for SSH configuration file                     [ Found ]
[10:30:49]   Checking if SSH root access is allowed                  [ Warning ]
[10:30:49] Warning: The SSH and rkhunter SSH configuration options are different.
[10:30:49]          SSH configuration option 'PermitRootLogin': prohibit-password
[10:30:49]          Rkhunter configuration option 'ALLOW_SSH_ROOT_USER': no
```

Common legitimate warnings to whitelist:
- Hidden directories in `/dev` — created by udev, normal
- SSH config mismatches — update `rkhunter.conf` to match your `sshd_config`
- Script warnings for `/usr/sbin/adduser`, `/usr/bin/ldd` — common false positives

```bash
# Whitelist a specific warning after verifying it is legitimate
# Edit /etc/rkhunter.conf and add to ALLOWHIDDENDIR or SCRIPTWHITELIST

# Rerun to confirm warning is gone
sudo rkhunter --check --skip-keypress 2>&1 | grep Warning
```

## Automating Both Scanners

```bash
# /usr/local/bin/rootkit-scan.sh — runs both scanners, emails summary

#!/bin/bash
REPORT=$(mktemp)
ADMIN="admin@yourdomain.com"
HOST=$(hostname)
DATE=$(date +%Y-%m-%d)

echo "=== Rootkit Scan Report: $HOST - $DATE ===" >> "$REPORT"

# chkrootkit
echo "" >> "$REPORT"
echo "--- chkrootkit ---" >> "$REPORT"
sudo chkrootkit -q 2>&1 >> "$REPORT"

# rkhunter
echo "" >> "$REPORT"
echo "--- rkhunter ---" >> "$REPORT"
sudo rkhunter --check --skip-keypress 2>&1 | \
  grep -E "Warning|Infected|Found|Rootkit" >> "$REPORT"

# Send only if something suspicious found
if grep -qiE "infected|warning|rootkit found" "$REPORT"; then
  mail -s "[ALERT] Rootkit scan: issues on $HOST" "$ADMIN" < "$REPORT"
else
  echo "Clean scan on $HOST: $DATE" | mail -s "Rootkit scan: clean $HOST" "$ADMIN"
fi

rm -f "$REPORT"
```

```bash
chmod +x /usr/local/bin/rootkit-scan.sh

# Add weekly cron job
echo "0 4 * * 0 root /usr/local/bin/rootkit-scan.sh" | \
  sudo tee /etc/cron.d/rootkit-scan
```

## If a Rootkit Is Found

Do not trust any tool on the compromised system. Assume the rootkit has modified `ls`, `ps`, `netstat`, and the kernel itself.

```bash
# Step 1: Isolate the system (block outbound connections)
sudo iptables -P OUTPUT DROP

# Step 2: Take a memory dump if possible (rootkit may be in RAM only)
# Boot from external media for this

# Step 3: Capture disk image for forensics
dd if=/dev/sda bs=4096 | gzip > /media/external/disk-image.gz

# Step 4: Do NOT attempt to clean — rebuild from a known-good image
# Cleaning a rootkit infection is unreliable

# Step 5: Investigate how the rootkit got in
# Check auth logs, web server logs, package logs
```

## Comparing the Two Tools

| Feature | chkrootkit | rkhunter |
|---------|-----------|----------|
| Rootkit signatures | 70+ | 400+ |
| Binary hash checks | No | Yes |
| Configuration file | No | Yes |
| Log file | No | Yes |
| Update mechanism | Manual | `--update` |
| False positive rate | Higher | Lower (with tuning) |

Run both. They use different detection methods and catch different things.

## Related Reading

- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
- [How to Use Tripwire for File Integrity Monitoring](/privacy-tools-guide/tripwire-file-integrity-monitoring-guide/)
- [Lynis Linux Security Audit Guide](/privacy-tools-guide/lynis-linux-security-audit-guide/)
- [Email Tracking Pixel Detection](/privacy-tools-guide/email-tracking-pixel-detection-how-to-identify-and-block-spy/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

## Related Articles

- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)
- [How to Use Lynis for Linux Security Auditing](/privacy-tools-guide/lynis-linux-security-audit-guide/)
- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
- [Setting Up Vault for Secrets Management](/privacy-tools-guide/hashicorp-vault-secrets-management-setup/)
- [Nextcloud Setup Guide Raspberry Pi 2026](/privacy-tools-guide/nextcloud-setup-guide-raspberry-pi-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
