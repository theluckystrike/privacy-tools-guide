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

## Monitoring Backup Success and Alerting on Failure

A backup that runs silently and fails silently is worse than no backup. Add monitoring to detect failures before you need to restore.

**Email notification on failure:**

```bash
#!/bin/bash
# Add to the backup script after the borg create block

MAILTO="alerts@example.com"
HOSTNAME=$(hostname -s)

if [ $? -ne 0 ]; then
    echo "Borg backup failed on $HOSTNAME at $(date)" | \
        mail -s "BACKUP FAILURE: $HOSTNAME" "$MAILTO"
    exit 1
fi
```

**Healthchecks.io integration** (free tier available — pings a URL to confirm backup ran):

```bash
# Add at the END of your backup script
# Create a check at healthchecks.io, get the ping URL
HEALTHCHECK_URL="https://hc-ping.com/your-uuid-here"

curl -fsS --retry 3 "${HEALTHCHECK_URL}" > /dev/null 2>&1
```

If the cron job does not run or the script exits early, the healthchecks.io check goes stale and sends an alert. No ping = something went wrong.

**Prometheus metrics for Borg:**

```bash
# After successful backup, write metrics to node_exporter textfile dir
STATS=$(borg info "${BORG_REPO}::${ARCHIVE_NAME}" --json 2>/dev/null)
ORIGINAL=$(echo "$STATS" | python3 -c "import sys,json; d=json.load(sys.stdin)['cache']['stats']; print(d['total_size'])")

cat > /var/lib/node_exporter/textfile_collector/borg.prom <<EOF
# HELP borg_backup_last_success_timestamp Unix timestamp of last successful backup
# TYPE borg_backup_last_success_timestamp gauge
borg_backup_last_success_timestamp $(date +%s)
# HELP borg_backup_original_size_bytes Original data size in bytes
# TYPE borg_backup_original_size_bytes gauge
borg_backup_original_size_bytes ${ORIGINAL}
EOF
```

Alert in Prometheus if `time() - borg_backup_last_success_timestamp > 86400` (no backup in 24 hours).

## Offsite Replication to Backblaze B2

Borg native repositories can be mirrored to Backblaze B2 using rclone after the local backup completes:

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure B2 backend
rclone config
# Type: b2, account ID: your_key_id, key: your_application_key

# Sync the Borg repo directory to B2
rclone sync /mnt/backup/myrepo \
  b2:my-backup-bucket/myhost \
  --fast-list \
  --transfers 4 \
  --b2-hard-delete

# Add to backup script after borg compact:
rclone sync "${LOCAL_BORG_DIR}" "b2:my-bucket/${HOSTNAME}" \
  --fast-list --transfers 4 >> "$LOG" 2>&1
```

Borg's data blocks are already AES-256 encrypted before rclone uploads them — Backblaze never sees plaintext data. This gives you:
- Local copy for fast restores
- Offsite encrypted copy in B2 for disaster recovery
- B2 versioning enabled at the bucket level for additional snapshot retention

Keep B2 costs low: enable B2's lifecycle rules to delete files older than your longest retention period.

## Related Articles

- [Encrypted Backup Solutions Comparison 2026](/encrypted-backup-solutions-comparison-2026/)
- [Restic vs Borg Backup: Encrypted Comparison for Developers](/restic-vs-borg-backup-encrypted-comparison/)
- [Best Encrypted Backup Solution For Developers](/best-encrypted-backup-solution-for-developers/)
- [Set Up Encrypted Local Backup Of iPhone](/how-to-set-up-encrypted-local-backup-of-iphone-without-using-icloud/)
- [Encrypted Backup Of Chat History How To Preserve Messages](/encrypted-backup-of-chat-history-how-to-preserve-messages-wi/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
