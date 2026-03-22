---
layout: default
title: "How to Use Lynis for Linux Security Auditing"
description: "Run Lynis security audits on Linux servers, interpret hardening index scores, fix common findings, and automate recurring scans in CI or cron"
date: 2026-03-22
author: theluckystrike
permalink: /lynis-linux-security-audit-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

# How to Use Lynis for Linux Security Auditing

Lynis is an open-source security auditing tool for Linux, macOS, and BSD systems. It checks hundreds of security controls — SSH config, kernel parameters, file permissions, installed software, authentication settings — and gives you a hardening index score. It runs locally as the user you specify (usually root), which means it can find misconfigurations that a remote scanner would miss.

## Installation

```bash
# Debian/Ubuntu
sudo apt install lynis

# RHEL/CentOS/Amazon Linux
sudo yum install lynis

# Or install from source for the latest version
git clone https://github.com/CISOfy/lynis
cd lynis
sudo ./lynis audit system
```

Always use the latest version — the check database is updated frequently.

```bash
lynis update info
lynis update release  # download latest if available
```

## Running a Full Audit

```bash
# Full system audit (requires root for some checks)
sudo lynis audit system

# Audit without root (skips privileged checks)
lynis audit system

# Write report to a custom path
sudo lynis audit system --report-file /var/log/lynis-report.dat

# Non-interactive mode — useful in scripts/CI
sudo lynis audit system --no-colors --quiet
```

The audit takes 2–5 minutes. Output is color-coded: green (OK), yellow (suggestion), red (warning).

## Understanding the Output

At the end of a run you see the **Hardening Index** — a score from 0 to 100.

```
-[ Lynis 3.1.2 Results ]-

  Warnings (2):
  ----------------------------
  ! Found some information disclosure in SMTP banner [MAIL-8818]
  ! CUPS daemon is not needed here [PRNT-2307]

  Suggestions (34):
  ----------------------------
  * Consider hardening SSH configuration [SSH-7408]
    - Details  : AllowTcpForwarding (YES --> NO)
  * Disable IPv6 if not needed [NETW-3015]
  * Enable process accounting [ACCT-9622]

  Follow-up:
  ----------------------------
  - Show details of a test           : lynis show details TEST-ID
  - Check the log file               : /var/log/lynis.log
  - Read security controls           : https://cisofy.com/lynis/controls/

  Lynis security scan details:
  Hardening index : 67 [#############       ]
  Tests performed : 248
  Plugins enabled : 0
```

A score of 67 is typical for a default Ubuntu/Debian install. Aim for 80+ on production systems.

## Fixing Common Warnings

### SSH Hardening (SSH-7408)

```bash
# Show what Lynis found for SSH
sudo lynis show details SSH-7408

# Edit /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config
```

Add or update these lines:

```
# Disable unused features
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
PermitRootLogin no

# Strong ciphers and MACs
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com

# Timeouts
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
MaxAuthTries 3

# Logging
LogLevel VERBOSE
```

```bash
# Test config before restarting
sudo sshd -t && sudo systemctl restart sshd
```

### Kernel Parameters (KRNL-6000)

```bash
# View kernel hardening suggestions
sudo lynis show details KRNL-6000

# Create a hardening sysctl file
sudo tee /etc/sysctl.d/99-hardening.conf << 'EOF'
# Disable IP forwarding unless this is a router
net.ipv4.ip_forward = 0

# Protect against SYN flood
net.ipv4.tcp_syncookies = 1

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0

# Log martian packets
net.ipv4.conf.all.log_martians = 1

# Prevent core dumps from leaking
fs.suid_dumpable = 0

# Restrict dmesg to root
kernel.dmesg_restrict = 1

# Restrict perf events
kernel.perf_event_paranoid = 3

# Disable magic SysRq
kernel.sysrq = 0

# Prevent ptrace across process boundaries
kernel.yama.ptrace_scope = 1

# Randomize memory layout (ASLR)
kernel.randomize_va_space = 2
EOF

sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

### Password Policy (AUTH-9282, AUTH-9286)

```bash
# Install password quality checking
sudo apt install libpam-pwquality   # Debian/Ubuntu
sudo yum install libpwquality       # RHEL

# Edit /etc/security/pwquality.conf
sudo tee /etc/security/pwquality.conf << 'EOF'
minlen = 14
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
maxrepeat = 3
usercheck = 1
EOF

# Set password aging
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   14/' /etc/login.defs
```

### File Permissions (FILE-7524)

```bash
# Find world-writable files and directories
sudo find / -path /proc -prune -o -path /sys -prune -o -perm -002 -type f -print

# Fix common issues
sudo chmod 700 /root
sudo chmod 600 /etc/crontab
sudo chmod 700 /etc/cron.d /etc/cron.daily /etc/cron.weekly /etc/cron.monthly
sudo chmod 600 /etc/ssh/sshd_config
```

## Comparing Scans Over Time

Lynis writes a machine-readable report to `/var/log/lynis-report.dat`.

```bash
# Parse the report for scores and warnings
grep -E "^(hardening_index|warning\[\]|suggestion\[\])" /var/log/lynis-report.dat | head -40

# Simple diff between two reports
diff /var/log/lynis-report-2026-03-01.dat /var/log/lynis-report.dat | grep "^[<>]"
```

## Automating with Cron

```bash
# /usr/local/bin/lynis-scheduled.sh
#!/bin/bash
set -euo pipefail

REPORT_DIR="/var/log/lynis"
DATE=$(date +%Y-%m-%d)
REPORT="${REPORT_DIR}/report-${DATE}.dat"
LOG="${REPORT_DIR}/log-${DATE}.txt"

mkdir -p "$REPORT_DIR"

/usr/bin/lynis audit system \
  --no-colors \
  --quiet \
  --report-file "$REPORT" \
  >> "$LOG" 2>&1

SCORE=$(grep "^hardening_index=" "$REPORT" | cut -d= -f2)
WARNINGS=$(grep "^warning\[\]=" "$REPORT" | wc -l)

echo "Lynis scan ${DATE}: score=${SCORE}, warnings=${WARNINGS}" | \
  mail -s "Lynis Audit ${DATE} - Score: ${SCORE}" admin@example.com || true

# Keep only last 30 reports
find "$REPORT_DIR" -name "report-*.dat" -mtime +30 -delete
find "$REPORT_DIR" -name "log-*.txt" -mtime +30 -delete
```

```bash
chmod +x /usr/local/bin/lynis-scheduled.sh

# Add weekly cron job
echo "0 3 * * 0 root /usr/local/bin/lynis-scheduled.sh" | \
  sudo tee /etc/cron.d/lynis-audit
```

## Running Lynis in a CI Pipeline

Use Lynis to gate deployments on golden images.

```yaml
# .github/workflows/image-audit.yml
name: Security Audit

on:
  push:
    paths:
      - 'packer/**'
      - 'ansible/**'

jobs:
  lynis-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Lynis
        run: |
          git clone --depth 1 https://github.com/CISOfy/lynis /opt/lynis

      - name: Run audit
        run: |
          sudo /opt/lynis/lynis audit system \
            --no-colors \
            --quiet \
            --report-file /tmp/lynis-report.dat

      - name: Check hardening index
        run: |
          SCORE=$(grep "^hardening_index=" /tmp/lynis-report.dat | cut -d= -f2)
          echo "Hardening index: $SCORE"
          if [ "$SCORE" -lt 75 ]; then
            echo "::error::Hardening index $SCORE is below threshold of 75"
            exit 1
          fi

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: lynis-report
          path: /tmp/lynis-report.dat
```

## Related Articles

- [How to Audit npm Packages for Security](/privacy-tools-guide/audit-npm-packages-security-guide/)
- [VPN Provider Annual Audit Results: Independent Security](/privacy-tools-guide/vpn-provider-annual-audit-results-independent-security-verified/)
- [Linux File Permissions Privacy](/privacy-tools-guide/linux-file-permissions-privacy-audit/)
- [Arch Linux Hardened Kernel Installation Guide For Advanced](/privacy-tools-guide/arch-linux-hardened-kernel-installation-guide-for-advanced-p/)
- [How to Audit Your Password Manager Vault: A Practical Guide](/privacy-tools-guide/how-to-audit-your-password-manager-vault/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
