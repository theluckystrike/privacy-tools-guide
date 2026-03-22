---
layout: default
title: "How to Use Tripwire for File Integrity Monitoring"
description: "Install Open Source Tripwire, build a baseline database, run integrity checks, interpret reports, and configure email alerts for unauthorized file changes"
date: 2026-03-22
author: theluckystrike
permalink: /tripwire-file-integrity-monitoring-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Use Tripwire for File Integrity Monitoring

Tripwire takes a cryptographic snapshot of your filesystem — hashing files, recording permissions, and noting ownership. When run again later, it compares the current state to that baseline and reports every change. This detects rootkit installations, unauthorized file modifications, and configuration drift.

## Installation

```bash
# Debian/Ubuntu
sudo apt install tripwire

# During installation, you are prompted for two passphrases:
# - Site key passphrase: protects the policy and configuration files
# - Local key passphrase: protects the database and report files

# RHEL/CentOS (Wazuh fork is commonly used here)
sudo yum install tripwire

# Or build from source
git clone https://github.com/Tripwire/tripwire-open-source
cd tripwire-open-source && mkdir build && cd build
cmake -DENABLE_TESTS=OFF .. && make -j$(nproc) && sudo make install
```

## Configuration Files

Tripwire has three main files:

| File | Purpose |
|------|---------|
| `/etc/tripwire/twcfg.txt` | Master configuration (paths, mail settings) |
| `/etc/tripwire/twpol.txt` | Policy: which files to monitor and how |
| `/var/lib/tripwire/*.twd` | The database (signed baseline) |

## Understanding the Policy File

The policy defines which files to monitor and what properties to check.

```
# Policy variable shortcuts — define groups of properties to check

# Most thorough: check everything
SEC_CRIT      = $(IgnoreNone)-SHa ; # Critical files (all attributes)

# Check everything except access time
SEC_SUID      = $(IgnoreNone)-SHa+u ; # setuid programs

# Only check existence and major properties
SEC_BIN       = $(ReadOnly) ;  # Binary files (read-only, changes are suspicious)

# Logging files: only check that they still exist, not content
SEC_LOG       = $(Growing) ;   # Log files (allow growth, not shrinkage)

# Configuration: check everything except size (some configs are dynamic)
SEC_CONFIG    = $(Dynamic) ;
```

```
# Example policy rules in twpol.txt

# Critical system directories
(
  rulename = "OS Binaries and Libraries",
  severity = $(SIG_HI)
)
{
  /bin                    -> $(SEC_BIN) ;
  /sbin                   -> $(SEC_BIN) ;
  /usr/bin                -> $(SEC_BIN) ;
  /usr/sbin               -> $(SEC_BIN) ;
  /lib                    -> $(SEC_BIN) ;
  /usr/lib                -> $(SEC_BIN) ;
}

# Configuration files
(
  rulename = "System Configuration",
  severity = $(SIG_HI)
)
{
  /etc/passwd             -> $(SEC_CRIT) ;
  /etc/shadow             -> $(SEC_CRIT) ;
  /etc/sudoers            -> $(SEC_CRIT) ;
  /etc/ssh/sshd_config    -> $(SEC_CRIT) ;
  /etc/hosts.allow        -> $(SEC_CONFIG) ;
  /etc/hosts.deny         -> $(SEC_CONFIG) ;
  /etc/crontab            -> $(SEC_CRIT) ;
  /etc/cron.d             -> $(SEC_CRIT) ;
}

# Web server content
(
  rulename = "Web Root",
  severity = $(SIG_MED)
)
{
  /var/www/html           -> $(SEC_CONFIG) ;
}

# Exceptions — files that change constantly
(
  rulename = "Ignore Volatile Files",
  severity = $(SIG_LOW)
)
{
  !/etc/mtab ;
  !/etc/adjtime ;
  !/etc/random-seed ;
  !/etc/ntp.drift ;
  !/proc ;
  !/sys ;
  !/dev ;
  !/run ;
  !/tmp ;
}
```

## Building the Initial Baseline

After editing the policy file, generate the signed database:

```bash
# First, sign the policy and config files with the site key
sudo twadmin --create-polfile /etc/tripwire/twpol.txt
sudo twadmin --create-cfgfile /etc/tripwire/twcfg.txt

# Initialize the database — this is the baseline snapshot
sudo tripwire --init

# Enter your local key passphrase when prompted
# The database is created at /var/lib/tripwire/$(hostname).twd
```

Run the initial check immediately after to confirm no false positives in the baseline:

```bash
sudo tripwire --check 2>&1 | tail -20
# "No violations."
```

If there are violations right away, the policy has rules that do not match your system — refine the policy, then re-run `--init`.

## Running an Integrity Check

```bash
# Manual check
sudo tripwire --check

# Check and output report to file
sudo tripwire --check --report-file /var/lib/tripwire/reports/$(hostname)-$(date +%Y%m%d).twr

# Read the report (rendered human-readable)
sudo twprint --print-report --twrfile /var/lib/tripwire/reports/*.twr | less
```

Report output interpretation:

```
Rule Name: System Configuration (/etc)
Severity Level: 100

Added:
  /etc/cron.d/newjob                 # New file — verify if expected

Modified:
  /etc/passwd                        # Change — check if user was added
    Attr:  mtime changed             # Modification time changed
    Attr:  size changed              # File size changed

Removed:
  (none)
```

## Responding to Violations

For each violation, investigate before updating the baseline:

```bash
# Check who changed a file and when
stat /etc/passwd
sudo last | head -20   # who logged in recently
sudo ausearch -f /etc/passwd  # if auditd is running

# Check git diff if config files are version controlled
git -C /etc diff /etc/passwd   # if you use etckeeper

# If the change is legitimate, update the database
sudo tripwire --update --accept-all --twrfile /var/lib/tripwire/reports/$(hostname)-latest.twr

# If you want to review each change interactively:
sudo tripwire --update --twrfile /var/lib/tripwire/reports/$(hostname)-latest.twr
```

## Automating with Cron and Email Alerts

```bash
# /etc/cron.d/tripwire-check
# Run daily at 3am, email report
0 3 * * * root /usr/sbin/tripwire --check --email-report > /dev/null 2>&1
```

Configure email in `/etc/tripwire/twcfg.txt`:

```
MAILNOVIOLATIONS    =true
MAILFROM            =tripwire@yourserver.example.com
SMTPHOST            =127.0.0.1
SMTPPORT            =25
```

Or write a custom script:

```bash
#!/bin/bash
# /usr/local/bin/tripwire-check.sh

REPORT_DIR="/var/lib/tripwire/reports"
REPORT="${REPORT_DIR}/$(hostname)-$(date +%Y%m%d).twr"
ADMIN_EMAIL="admin@yourdomain.com"

# Run check
sudo tripwire --check --report-file "$REPORT" > /dev/null 2>&1

# Parse violations
VIOLATIONS=$(sudo twprint --print-report --twrfile "$REPORT" 2>/dev/null \
  | grep -c "Modified:\|Added:\|Removed:")

if [ "$VIOLATIONS" -gt 0 ]; then
  echo "Subject: [ALERT] Tripwire violations on $(hostname)" | \
  cat - <(sudo twprint --print-report --twrfile "$REPORT" | head -100) | \
  sendmail "$ADMIN_EMAIL"
fi
```

## Protecting the Database

The Tripwire database is signed with your local key — without the passphrase, an attacker cannot forge a clean database. But they could delete it. Store a copy offline:

```bash
# Backup the signed database after each successful init or update
cp /var/lib/tripwire/$(hostname).twd /root/tripwire-backup/$(hostname)-$(date +%Y%m%d).twd

# Optionally copy to a remote server
rsync /var/lib/tripwire/*.twd backup-user@backup-server:/var/tripwire-backups/
```

Also store the policy, config, and keys:

```bash
tar czf /root/tripwire-keys-$(date +%Y%m%d).tar.gz \
  /etc/tripwire/site.key \
  /etc/tripwire/$(hostname)-local.key \
  /etc/tripwire/tw.pol \
  /etc/tripwire/tw.cfg

# Encrypt the archive
gpg -c /root/tripwire-keys-$(date +%Y%m%d).tar.gz
```

## Related Reading

- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
- [How to Use chkrootkit and rkhunter](/privacy-tools-guide/chkrootkit-rkhunter-rootkit-detection-guide/)
- [Lynis Linux Security Audit Guide](/privacy-tools-guide/lynis-linux-security-audit-guide/)
- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

## Related Articles

- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)
- [Privacy Focused File Transfer Tools Comparison 2026](/privacy-tools-guide/privacy-focused-file-transfer-tools-comparison-2026/)
- [Magic Wormhole Encrypted File Transfer How To Send Files](/privacy-tools-guide/magic-wormhole-encrypted-file-transfer-how-to-send-files-sec/)
- [Secure File Sharing Tools Comparison: E2E Encrypted](/privacy-tools-guide/secure-file-sharing-tools-comparison/)
- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
