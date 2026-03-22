---
layout: default
title: "How to Use BorgBackup for Encrypted Backups"
description: "Set up BorgBackup with AES-256 encryption, deduplication, and automated offsite backup to a remote SSH server or cloud storage"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-use-borgbackup-for-encrypted-backups/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

BorgBackup (Borg) is a deduplicating backup tool with built-in AES-256-CTR encryption. It compresses and deduplicates data at the chunk level, so backups after the first are fast and space-efficient. This guide covers creating an encrypted repo, automating backups, pruning old snapshots, and restoring files.

## Installation

```bash
# Ubuntu/Debian
sudo apt install borgbackup

# macOS
brew install borgbackup

# Verify
borg --version
# borg 1.4.x
```

## Step 1: Initialize an Encrypted Repository

A Borg repository is a directory (local or remote) that stores all backup data.

```bash
# Local repository
borg init --encryption=repokey-blake2 /mnt/backup/myrepo

# Remote repository over SSH
borg init --encryption=repokey-blake2 user@backupserver:/path/to/repo

# You will be prompted to set a passphrase — store this in a password manager
```

`repokey-blake2` stores the encrypted key inside the repository, protected by your passphrase. Export and store your repo key offline:

```bash
borg key export /mnt/backup/myrepo ~/borg-key-backup.txt
```

## Step 2: Create Your First Backup

```bash
borg create \
  --verbose \
  --stats \
  --compression lz4 \
  /mnt/backup/myrepo::backup-{now:%Y-%m-%d} \
  ~/Documents \
  ~/Projects \
  ~/Pictures
```

Common compression options: `lz4` (fast), `zstd,3` (better ratio), `none` (pre-compressed files).

Exclude patterns:

```bash
borg create \
  --verbose \
  --stats \
  --compression zstd,3 \
  --exclude-caches \
  --exclude '*.pyc' \
  --exclude '**/.git' \
  --exclude '**/node_modules' \
  /mnt/backup/myrepo::backup-{now:%Y-%m-%dT%H:%M} \
  /home/user
```

## Step 3: List and Browse Archives

```bash
# List all archives
borg list /mnt/backup/myrepo

# List contents of a specific archive
borg list /mnt/backup/myrepo::backup-2026-03-22

# Search for a specific file
borg list /mnt/backup/myrepo --glob-archives '*' | head -20
```

## Step 4: Restore Files

```bash
# Restore an entire archive
mkdir /tmp/restore && cd /tmp/restore
borg extract /mnt/backup/myrepo::backup-2026-03-22

# Restore a specific path only
borg extract /mnt/backup/myrepo::backup-2026-03-22 \
  home/user/Documents/important.pdf

# Dry run
borg extract --dry-run --list /mnt/backup/myrepo::backup-2026-03-22
```

## Step 5: Prune Old Archives

```bash
borg prune \
  --verbose \
  --list \
  --keep-hourly 24 \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --keep-yearly 2 \
  /mnt/backup/myrepo

# Compact to reclaim disk space
borg compact /mnt/backup/myrepo
```

## Step 6: Automate with a Shell Script

```bash
#!/bin/bash
# /usr/local/bin/borg-backup.sh

export BORG_REPO="user@backupserver:/backups/myhost"
export BORG_PASSPHRASE="your-secure-passphrase"
export BORG_RSH="ssh -i /root/.ssh/borg_backup_key"

LOG="/var/log/borg-backup.log"

info() { echo "$(date) [INFO] $*" >> "$LOG"; }
error() { echo "$(date) [ERROR] $*" >> "$LOG"; }

info "Starting backup"

borg create \
  --verbose \
  --stats \
  --compression zstd,3 \
  --exclude-caches \
  --exclude '/home/*/.cache' \
  --exclude '/tmp' \
  "${BORG_REPO}::backup-{now:%Y-%m-%dT%H:%M}" \
  /home \
  /etc \
  /var/www \
  2>> "$LOG"

if [ $? -eq 0 ]; then
    info "Backup completed successfully"
else
    error "Backup FAILED"
    exit 1
fi

borg prune \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  "${BORG_REPO}" >> "$LOG" 2>&1

borg compact "${BORG_REPO}" >> "$LOG" 2>&1
info "Done"
```

Add to crontab:
```bash
sudo chmod +x /usr/local/bin/borg-backup.sh
echo "0 2 * * * root /usr/local/bin/borg-backup.sh" | \
  sudo tee /etc/cron.d/borg-backup
```

## Step 7: Remote Backup SSH Setup

Create a dedicated SSH key and restrict it on the backup server:

```bash
# On backup client
ssh-keygen -t ed25519 -f ~/.ssh/borg_backup_key -N ""

# On backup server — restrict key to borg only
cat >> ~/.ssh/authorized_keys << 'EOF'
command="borg serve --restrict-to-path /backups/myhost",restrict ssh-ed25519 AAAA... borg@client
EOF
```

## Step 8: Verify Backup Integrity

```bash
# Check repository consistency
borg check /mnt/backup/myrepo

# Verify checksums of all data
borg check --verify-data /mnt/backup/myrepo
```

Run `borg check` at least monthly. A backup you have never verified may not be restorable.

## Related Articles

- [Encrypted Backup Solutions Comparison 2026](/privacy-tools-guide/encrypted-backup-solutions-comparison-2026/)
- [Restic vs Borg Backup: Encrypted Comparison for Developers](/privacy-tools-guide/restic-vs-borg-backup-encrypted-comparison/)
- [Best Encrypted Backup Solution For Developers](/privacy-tools-guide/best-encrypted-backup-solution-for-developers/)
- [Set Up Encrypted Local Backup Of iPhone](/privacy-tools-guide/how-to-set-up-encrypted-local-backup-of-iphone-without-using-icloud/)
- [Encrypted Backup Of Chat History How To Preserve Messages](/privacy-tools-guide/encrypted-backup-of-chat-history-how-to-preserve-messages-wi/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
