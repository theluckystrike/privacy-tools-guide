---
layout: default
title: "Linux File Permissions Privacy Audit"
description: "How to audit Linux file permissions to find world-readable sensitive files, SUID binaries, insecure home directories, and misconfigured shared directories"
date: 2026-03-21
author: theluckystrike
permalink: /linux-file-permissions-privacy-audit/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Linux file permissions control who can read, write, and execute every file on your system. Misconfigured permissions are one of the most common vectors for local privilege escalation and data exposure: a world-readable private key, a group-writable script in a cron job, or a home directory accessible to other users can each lead to data breaches or account compromise.

This guide walks through auditing your Linux system for permission-related privacy and security issues.

## Understanding Linux Permissions

Every file and directory has three permission sets: owner, group, and others. Each set has three bits: read (r=4), write (w=2), execute (x=1).

```
-rwxr-xr-- 1 alice developers 4096 Mar 21 file.sh
 ^^^       ← owner (alice): read, write, execute
    ^^^    ← group (developers): read, execute
       ^^^ ← others: read only
```

In octal: `754`

**The most dangerous permissions:**
- `777` — anyone can read, write, execute
- `o+w` — others can write (anyone on the system can modify the file)
- `o+r` on sensitive files — anyone can read private data
- SUID bit (`4xxx`) on unusual executables — runs as file owner (often root)

## Step 1: Audit Home Directory Permissions

Other users on the system should not be able to browse your home directory:

```bash
# Check permissions on all home directories
ls -la /home/

# Correct permission for a home directory: 700 (owner only) or 750 (owner + group)
# Incorrect: 755 (others can list and read world-readable files)

# Fix permissions on your home directory
chmod 700 /home/$USER

# Check for world-readable files in your home directory
find /home/$USER -perm /o+r -type f 2>/dev/null | head -30

# Check for world-readable directories
find /home/$USER -perm /o+r -type d 2>/dev/null | head -20
```

## Step 2: Find World-Writable Files

World-writable files can be modified by any user on the system — a vector for privilege escalation if the file is executed by a higher-privileged user:

```bash
# Find all world-writable files (excluding /proc, /sys, /dev)
find / \( -path /proc -o -path /sys -o -path /dev \) -prune -o \
  -perm -0002 -type f -print 2>/dev/null

# Focus on high-risk directories
find /etc /usr/local /opt -perm -0002 -type f 2>/dev/null
find /var/www /srv -perm -0002 -type f 2>/dev/null

# World-writable directories are also risks (anyone can add files)
find / \( -path /proc -o -path /sys -o -path /dev -o -path /tmp \) -prune -o \
  -perm -0002 -type d -print 2>/dev/null | grep -v "^/tmp"
```

Any world-writable file outside of `/tmp` and `/var/tmp` is unusual and should be investigated.

## Step 3: Audit SUID and SGID Binaries

SUID binaries run as the file owner (usually root) regardless of who executes them. Legitimate SUID binaries include `passwd`, `sudo`, and `ping`. Unknown SUID binaries are red flags:

```bash
# Find all SUID binaries on the system
find / \( -path /proc -o -path /sys -o -path /dev \) -prune -o \
  -perm -4000 -type f -print 2>/dev/null

# Find all SGID binaries
find / \( -path /proc -o -path /sys -o -path /dev \) -prune -o \
  -perm -2000 -type f -print 2>/dev/null

# Legitimate SUID files (vary by distro — learn your baseline)
# Common: /usr/bin/passwd, /usr/bin/sudo, /usr/bin/su, /bin/ping

# Compare against known-good list
find /usr/bin /usr/sbin /bin /sbin -perm -4000 -type f | sort
```

If you find unexpected SUID binaries — particularly in `/tmp`, home directories, or non-standard paths — investigate immediately. These are common persistence mechanisms for attackers.

## Step 4: Check SSH Key Permissions

SSH requires strict key file permissions. Keys with loose permissions are rejected by ssh:

```bash
# Check your SSH private key permissions (should be 600)
ls -la ~/.ssh/

# Correct permissions:
# ~/.ssh/           700  (owner rwx, no others)
# ~/.ssh/id_ed25519  600  (owner rw, no others)
# ~/.ssh/id_ed25519.pub 644 (owner rw, others r — public key, safe)
# ~/.ssh/authorized_keys 600 or 644
# ~/.ssh/config      600 or 644

# Fix if wrong
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/config

# Find SSH keys with wrong permissions across the system
find /home -name "id_rsa" -o -name "id_ed25519" -o -name "*.pem" 2>/dev/null | \
  xargs ls -la 2>/dev/null | grep -v "^-rw-------"
```

## Step 5: Audit Configuration File Permissions

Configuration files often contain credentials — database passwords, API keys, tokens:

```bash
# Find world-readable files in /etc that shouldn't be
find /etc -perm /o+r -type f 2>/dev/null | \
  grep -v "^/etc/alternatives\|^/etc/cron\|^/etc/apt\|^/etc/bash"

# Check specific sensitive files
ls -la /etc/shadow           # Should be 640 or 000
ls -la /etc/gshadow          # Should be 640 or 000
ls -la /etc/sudoers          # Should be 440
ls -la /etc/passwd           # 644 (world-readable, but no passwords)

# Find files containing typical credential patterns
grep -r "password\s*=" /etc/ --include="*.conf" --include="*.ini" 2>/dev/null | \
  grep -v "^Binary" | \
  while read file; do
    perm=$(stat -c "%a" "$(echo $file | cut -d: -f1)" 2>/dev/null)
    echo "$perm $file"
  done | grep "^[67]" # Flag if owner can't read (unusual) or group/other readable
```

## Step 6: Check Web Server File Permissions

If you run a web server, world-readable configuration files or uploaded content can expose credentials:

```bash
# Check Nginx/Apache configuration
find /etc/nginx /etc/apache2 -perm /o+r -type f 2>/dev/null
ls -la /etc/nginx/sites-available/

# Check for PHP config files (often contain database credentials)
find /var/www -name "*.php" -perm -o+r 2>/dev/null | head -20
find /var/www -name "wp-config.php" -o -name ".env" 2>/dev/null | xargs ls -la 2>/dev/null

# Database configuration should not be world-readable
find /var/www -name "database.php" -o -name "db_config.php" 2>/dev/null | \
  xargs ls -la 2>/dev/null | grep "r--"
```

## Step 7: Audit Cron Job Files

Cron jobs executed by privileged users that call world-writable scripts are privilege escalation vectors:

```bash
# Check permissions of files referenced in cron jobs
crontab -l 2>/dev/null
sudo crontab -l 2>/dev/null
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/

# Find world-writable scripts in cron directories
find /etc/cron* /var/spool/cron -perm -0002 2>/dev/null

# Check root's cron jobs reference files with safe permissions
sudo crontab -l 2>/dev/null | awk '{print $NF}' | xargs -I{} ls -la {} 2>/dev/null
```

## Step 8: Generate a Permissions Baseline

Create a snapshot of current SUID binaries and sensitive file permissions for future comparison:

```bash
#!/bin/bash
# /usr/local/bin/perms-baseline
# Run periodically and compare output to detect changes

echo "=== SUID Binaries ==="
find / \( -path /proc -o -path /sys -o -path /dev \) -prune -o \
  -perm -4000 -type f -print 2>/dev/null | sort

echo ""
echo "=== World-Writable Files (excl. /tmp) ==="
find / \( -path /proc -o -path /sys -o -path /dev -o -path /tmp \) -prune -o \
  -perm -0002 -type f -print 2>/dev/null | sort

echo ""
echo "=== SSH Key Permissions ==="
find /home -name "id_*" -not -name "*.pub" 2>/dev/null | xargs ls -la 2>/dev/null

echo ""
echo "=== Shadow File Permissions ==="
ls -la /etc/shadow /etc/gshadow /etc/sudoers 2>/dev/null
```

```bash
sudo chmod +x /usr/local/bin/perms-baseline
sudo /usr/local/bin/perms-baseline > /var/log/perms-baseline-$(date +%Y%m%d).txt

# Compare future runs against the baseline
diff /var/log/perms-baseline-20260321.txt \
     <(sudo /usr/local/bin/perms-baseline)
```

Any new SUID binaries or newly world-writable files since the baseline are worth investigating.

## Automated Auditing Tools

```bash
# lynis — comprehensive system audit
sudo apt install lynis
sudo lynis audit system

# Specific checks
sudo lynis audit system --tests-from-group file_permissions

# rkhunter — rootkit and file permission checks
sudo apt install rkhunter
sudo rkhunter --checkall
```

## Related Reading

- [SSH Server Hardening Guide](/ssh-server-hardening-guide/)
- [How to Set Up a Privacy-Focused Home Server](/how-to-set-up-secure-home-server-for-self-hosting-privacy-tools/)
- [Suricata Home Network IDS Setup Guide](/suricata-home-network-ids-setup/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
