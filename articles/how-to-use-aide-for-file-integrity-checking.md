---
layout: default
title: "How to Use AIDE for File Integrity Checking"
description: "Set up AIDE to create a cryptographic baseline of your filesystem, detect unauthorized changes, and alert on tampering in security-sensitive environments"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-use-aide-for-file-integrity-checking/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

AIDE (Advanced Intrusion Detection Environment) creates a cryptographic database of file attributes — hashes, permissions, ownership, modification times — and compares the current filesystem state against this baseline on subsequent runs. Any deviation indicates a potential compromise. Unlike real-time monitoring tools, AIDE is a forensic baseline that catches changes even from rootkits that suppress filesystem events.

## Installation

```bash
# Ubuntu/Debian
sudo apt install aide aide-common

# CentOS/RHEL
sudo dnf install aide

# Verify
aide --version
```

## Configuration Overview

The main configuration file is `/etc/aide/aide.conf`. It defines:
1. **Macros**: shorthand for sets of file attributes to check
2. **Rules**: which directories to monitor and with which macros
3. **Database paths**: where to store and read the baseline

```bash
sudo cat /etc/aide/aide.conf | head -60
```

## Customize the Configuration

AIDE's default config covers most system paths. Add your custom paths:

```bash
sudo tee /etc/aide/aide.conf.d/99-custom.conf > /dev/null <<'EOF'
# Monitor web application directories
/var/www/html CONTENT_EX

# Monitor application configuration (not logs)
/etc/nginx PERMS+CONTENT_EX
/etc/apache2 PERMS+CONTENT_EX
/etc/ssh PERMS+CONTENT_EX

# Monitor cron directories (persistence mechanism)
/etc/cron.d PERMS+CONTENT_EX
/etc/crontab PERMS+CONTENT_EX
/var/spool/cron PERMS+CONTENT_EX

# Monitor user sudoers
/etc/sudoers PERMS+CONTENT_EX
/etc/sudoers.d PERMS+CONTENT_EX

# Exclude frequently changing directories to reduce noise
!/var/log
!/var/cache
!/proc
!/sys
!/dev
!/run
!/tmp
!/var/tmp
EOF
```

AIDE attribute groups (common built-in macros):
| Macro | Checks |
|-------|--------|
| `NORMAL` | Permissions, inode, UID, GID, SHA256, SHA512 |
| `PERMS` | Permissions, inode, UID, GID only |
| `CONTENT` | SHA256, SHA512 only (ignores metadata) |
| `CONTENT_EX` | CONTENT + file type, permissions |

## Initialize the Baseline Database

**Run this immediately after a fresh, known-good system setup** — before any potential compromise:

```bash
# Initialize the database (takes 10-30 minutes on a full system)
sudo aide --init

# This creates:
# /var/lib/aide/aide.db.new

# Verify the database was created
ls -lh /var/lib/aide/aide.db.new

# Move it into place as the reference database
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

**Store a copy of `aide.db` offline** — an attacker who compromises the system may also modify the database to hide their tracks. Copy it to an external drive or encrypted remote storage immediately.

```bash
# Secure copy to remote storage
scp /var/lib/aide/aide.db backup@offsite:/aide-baselines/aide.db.$(hostname).$(date +%Y%m%d)
```

## Running a Check

```bash
# Compare current filesystem to baseline
sudo aide --check

# Verbose output
sudo aide --check --verbose

# Output to a report file
sudo aide --check 2>&1 | tee /var/log/aide/aide-check-$(date +%Y%m%d).log
```

Sample output when changes are found:

```
AIDE found differences between database and filesystem!!

Start timestamp: 2026-03-22 14:32:01 +0000 (AIDE 0.18.3)

Summary:
  Total number of files:  28541
  Added files:            2
  Removed files:          0
  Changed files:          3

---------------------------------------------------
Added files:
---------------------------------------------------

added: /var/www/html/shell.php
added: /usr/local/bin/backdoor

---------------------------------------------------
Changed files:
---------------------------------------------------

changed: /etc/passwd
Permissions: -rw-r--r-- | -rw-rw-r--
Inode: 131074 | 131074
SHA256: 9f86d08188... | a665a45920...

changed: /usr/bin/ls
SHA256: 3fc4ccfe74... | 1a2b3c4d5e...
```

The third changed file (`/usr/bin/ls` with a changed SHA256) suggests a trojanized binary — a serious finding that warrants incident response.

## Automating Daily Checks

```bash
# Create a wrapper script
sudo tee /usr/local/bin/aide-daily-check.sh > /dev/null <<'SCRIPT'
#!/bin/bash

LOG_DIR="/var/log/aide"
LOG_FILE="${LOG_DIR}/aide-$(date +%Y%m%d-%H%M%S).log"
ALERT_EMAIL="security@example.com"

mkdir -p "$LOG_DIR"

# Run check
aide --check > "$LOG_FILE" 2>&1
EXIT_CODE=$?

# 0 = no changes, 1 = changes found, 2 = errors
if [ $EXIT_CODE -eq 1 ]; then
    echo "AIDE detected filesystem changes on $(hostname) at $(date)" | \
      mail -s "AIDE Alert: $(hostname)" "$ALERT_EMAIL" < "$LOG_FILE" 2>/dev/null || \
      logger -p auth.crit "AIDE ALERT: filesystem changes detected — see $LOG_FILE"
elif [ $EXIT_CODE -gt 1 ]; then
    logger -p daemon.error "AIDE check failed with exit code $EXIT_CODE"
fi

# Compress old logs
find "$LOG_DIR" -name "aide-*.log" -mtime +7 -exec gzip {} \;
SCRIPT

sudo chmod +x /usr/local/bin/aide-daily-check.sh

# Add to cron
echo "30 3 * * * root /usr/local/bin/aide-daily-check.sh" | \
  sudo tee /etc/cron.d/aide-daily
```

Run the check at 3:30 AM daily when the system is less active.

## Updating the Database After Legitimate Changes

After planned software updates (e.g., `apt upgrade`), update the AIDE database so the new state becomes the new baseline:

```bash
# After running apt upgrade
sudo apt upgrade

# Re-initialize with new baseline
sudo aide --update

# This creates /var/lib/aide/aide.db.new
# Review the diff first
sudo aide --check

# If changes look expected (package updates only), accept the new baseline
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Archive the old database before overwriting
sudo cp /var/lib/aide/aide.db \
  /var/lib/aide/aide.db.$(date +%Y%m%d).bak
```

## AIDE vs inotifywait

AIDE provides cryptographic verification that survives system reboots and can be compared against an offline copy. inotifywait provides real-time notification. For serious security environments, use both:

1. **AIDE**: daily forensic baseline check, detects rootkit-level changes that may suppress inotify events
2. **inotifywait**: real-time alerting for immediate response to in-progress attacks

## Related Reading

- [How to Monitor File Changes with inotifywait](/privacy-tools-guide/how-to-monitor-file-changes-with-inotifywait/)
- [How to Detect Keyloggers on Your System](/privacy-tools-guide/how-to-detect-keyloggers-on-your-system/)
- [How to Use BorgBackup for Encrypted Backups](/privacy-tools-guide/how-to-use-borgbackup-for-encrypted-backups/)
- [How to Use Tripwire for File Integrity Monitoring](/privacy-tools-guide/tripwire-file-integrity-monitoring-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

## Related Articles

- [How to Use Tripwire for File Integrity Monitoring](/privacy-tools-guide/tripwire-file-integrity-monitoring-guide/)
- [Linux File Permissions Privacy](/privacy-tools-guide/linux-file-permissions-privacy-audit/)
- [Verify Your Devices Are Not Compromised: A Complete](/privacy-tools-guide/how-to-verify-your-devices-are-not-compromised-complete-audit/)
- [How to Set Up Promtail for Log Shipping](/privacy-tools-guide/how-to-set-up-promtail-for-log-shipping/)
- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
