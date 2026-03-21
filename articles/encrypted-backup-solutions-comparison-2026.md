---
layout: default
title: "Encrypted Backup Solutions Comparison 2026"
description: "Compare encrypted backup tools: Borg, Restic, Duplicati, Arq, Kopia. CLI examples, cloud storage integration, pricing, restore testing"
date: 2026-03-20
author: "Privacy Tools Guide"
permalink: /encrypted-backup-solutions-comparison-2026/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Regular backups protect against ransomware, hardware failure, and accidental deletion. Encrypted backups additionally protect your privacy: even if a backup is stolen, the data remains unreadable without the encryption key. This guide compares five encrypted backup solutions with different architectures, use cases, and pricing models.

## Why Encrypted Backups Matter

Standard backups without encryption expose your data:

- Backups stored on cloud services are readable by employees with access
- Ransomware targeting cloud backups can succeed without encryption
- Hard drives can be stolen with all data intact
- Data breaches expose backup copies of sensitive files

Encrypted backups ensure that even if someone accesses the backup, they can't read the data without your encryption key.

## Borg Backup: Deduplication-Focused

Borg is an open-source backup tool emphasizing efficiency through deduplication. Only stores changed data, dramatically reducing storage needs.

### Installation

```bash
# macOS
brew install borgbackup

# Linux (Ubuntu/Debian)
sudo apt install borgbackup

# Or via pip (all platforms)
pip install borgbackup
```

### Basic Setup: Local Backup

```bash
# Initialize backup repository
borg init --encryption=repokey ~/.backups/myrepo

# Create first backup
borg create ~/.backups/myrepo::backup-2026-03-20 \
  ~/Documents \
  ~/Pictures \
  --exclude '*.tmp' \
  --exclude '.git'

# List backups
borg list ~/.backups/myrepo

# Restore a file
borg extract ~/.backups/myrepo::backup-2026-03-20 Documents/important.txt
```

### Remote Backup to SSH Server

Backup to a remote server via SSH:

```bash
# Initialize remote repository
borg init --encryption=repokey ssh://user@backupserver:~/borg-repo

# Backup to remote
borg create ssh://user@backupserver:~/borg-repo::backup-2026-03-20 ~/Documents

# Restore from remote
borg extract ssh://user@backupserver:~/borg-repo::backup-2026-03-20
```

### Deduplication Example

Borg's efficiency through deduplication:

```
Backup 1: 100GB (initial backup)
- Documents: 50GB
- Photos: 50GB

Backup 2: 102GB total, but only 5GB new data
- Same documents and photos as Backup 1
- 2GB of new photos, 3GB of modified documents
- Borg stores: only the 5GB of changes
- Physical storage: ~105GB total, not 202GB

Backup 3: 103GB total, but only 2GB new data
- 1GB new photos, 1GB modified files
- Physical storage: ~107GB total
```

After 10 backups of the same data with incremental changes, you've stored 120GB of data in ~150GB of storage instead of 1.2TB.

### Automation: Cron Job

```bash
#!/bin/bash
# Daily backup script

export BORG_PASSPHRASE="your-encryption-passphrase"

borg create \
  --progress \
  --stats \
  ssh://user@server:~/borg-repo::backup-$(date +%Y-%m-%d) \
  ~/Documents \
  ~/Pictures \
  ~/Code \
  --exclude '*.tmp' \
  --exclude '.cache'

# Prune old backups (keep last 30 daily, 12 monthly, 2 yearly)
borg prune ssh://user@server:~/borg-repo \
  --keep-daily=30 \
  --keep-monthly=12 \
  --keep-yearly=2

# Check backup integrity
borg check ssh://user@server:~/borg-repo
```

Add to crontab:

```bash
# Daily backup at 2 AM
0 2 * * * /home/user/backup-script.sh
```

### Strengths

- Deduplication dramatically reduces storage
- Excellent for frequent backups (daily)
- Strong encryption (AES-256)
- Works with any SSH server (low-cost remote storage)
- Free and open-source
- Fast incremental backups

### Limitations

- Command-line only (no GUI)
- Steeper learning curve
- Single repository format (not compatible with other tools)
- Requires SSH access for remote backups
- No built-in cloud storage integrations

### Pricing

Free and open-source. Storage costs depend on destination (SSH server, self-hosted NAS, etc).

## Restic: Simplicity and Flexibility

Restic prioritizes ease of use while supporting multiple backends (local, SSH, S3, B2, Google Drive, etc).

### Installation

```bash
# macOS
brew install restic

# Linux
sudo apt install restic

# Or download binary from restic.net
```

### Basic Backup

```bash
# Initialize repository
restic init --repo /mnt/backups

# Create backup
restic -r /mnt/backups backup ~/Documents ~/Pictures

# List backups
restic -r /mnt/backups snapshots

# Restore files
restic -r /mnt/backups restore latest --target ~/restore-location
```

### Cloud Storage Integration

Restic works with S3 (AWS, Wasabi, DigitalOcean Spaces):

```bash
# Initialize on AWS S3
export AWS_ACCESS_KEY_ID=your-key
export AWS_SECRET_ACCESS_KEY=your-secret
restic init --repo s3:s3.amazonaws.com/my-backup-bucket

# Backup to S3
restic -r s3:s3.amazonaws.com/my-backup-bucket backup ~/Documents

# Restore from S3
restic -r s3:s3.amazonaws.com/my-backup-bucket restore latest
```

### Forget Policy (Retention)

```bash
# Keep last 7 daily, 4 weekly, 12 monthly backups
restic -r /mnt/backups forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

# Check what will be deleted before pruning
restic -r /mnt/backups forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --dry-run
```

### Automation: Systemd Timer

```ini
# /etc/systemd/system/restic-backup.service
[Unit]
Description=Restic Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/restic -r /mnt/backups backup \
  /home/user/Documents \
  /home/user/Pictures
Environment=RESTIC_PASSWORD=your-password

# /etc/systemd/system/restic-backup.timer
[Unit]
Description=Restic Backup Timer

[Timer]
OnCalendar=daily
OnCalendar=02:00

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl enable restic-backup.timer
sudo systemctl start restic-backup.timer
```

### Strengths

- Very simple commands and clear output
- Works with many backends (S3, SFTP, local, etc)
- Efficient deduplication (similar to Borg)
- Good documentation and community
- Restore is straightforward
- Free and open-source

### Limitations

- No built-in scheduling (need external tools)
- Less optimization for certain use cases
- Smaller ecosystem than commercial tools

### Pricing

Free and open-source. Cloud storage costs depend on provider.

## Duplicati: User-Friendly GUI

Duplicati provides a web-based interface making backups accessible to non-technical users while supporting encryption.

### Installation and Setup

```bash
# macOS
brew install duplicati

# Linux
sudo apt install duplicati

# Windows: download installer from duplicati.com

# Start the service
duplicati --start-tray
# Opens http://localhost:8200 in browser
```

### GUI Workflow

1. Open http://localhost:8200
2. Click "Add backup"
3. Choose destination (local drive, S3, Backblaze B2, Google Drive, etc)
4. Select folders to backup
5. Set encryption password
6. Choose schedule (daily, weekly, monthly)
7. Save

### Real-World Setup: Backup to AWS S3

```
GUI Steps:

1. New backup
2. Destination: Amazon S3
3. Enter AWS Access Key and Secret Key
4. Bucket: "my-family-backups"
5. Folders to backup:
   - /home/user/Documents
   - /home/user/Pictures
   - /home/user/Desktop
6. Encryption:
   - Passphrase: [strong password]
   - AES-256
7. Schedule: Daily at 2 AM
8. Backup: 100GB of data
9. Retention: Keep last 30 daily backups

Click Save and Duplicati handles everything.
```

### Advanced Configuration

Duplicati can be configured via command-line for automation:

```bash
duplicati-cli backup \
  "s3://aws_keyid:aws_secret@s3.amazonaws.com/bucket/path" \
  /path/to/backup \
  --passphrase "your-password" \
  --keep-time "30d" \
  --backup-name "my-backup"
```

### Strengths

- Easiest GUI of all options
- Supports many cloud backends
- Good retention policies
- Version history (restore previous versions)
- Good for non-technical users
- Free and open-source

### Limitations

- Slower performance than Borg or Restic
- Less efficient storage usage (less optimal deduplication)
- Can be resource-intensive for large backups
- Smaller community than commercial options

### Pricing

Free and open-source. Cloud storage costs based on provider.

## Arq: Commercial Backup Solution

Arq provides a polished macOS/Windows application with automatic backups and cloud integration.

### Installation and Setup

```
1. Download from arqbackup.com
2. Install and launch
3. Choose destination:
   - Amazon S3
   - Backblaze B2 (often $0.01/GB/month)
   - Google Cloud Storage
   - Dropbox, Box, OneDrive, etc
4. Select folders to backup
5. Set encryption password
6. Enable automatic backup (every 5 minutes)
```

### Features

**Continuous backup**: Monitors folders and backs up changes automatically.

**De-duplication**: Stores only changed files and blocks.

**Versioning**: Keep full history of file changes.

**Selective restore**: Restore individual files or full folders.

### Real-World Example: Photographer Workflow

```
Photographer backing up photo library:

Arq Settings:
- Source: /Users/photographer/Photos (500GB)
- Destination: Backblaze B2
- Encryption: AES-256
- Automatic backup every 30 minutes while working

Cost: 500GB on B2 = $5/month storage
Versioning: Keep 30 versions of each file

Scenario: Photographer accidentally deletes recent photos
- Open Arq
- Select deleted photos date range
- Restore to original location
- Photos recovered instantly
```

### Strengths

- Excellent macOS integration
- Automatic continuous backup
- Simple setup
- Reliable restoration
- Good customer support
- Works with local and cloud storage

### Limitations

- macOS/Windows only (no Linux)
- More expensive than open-source alternatives
- Fewer storage backend options than Duplicati
- Overkill for many use cases

### Pricing

$69.99 one-time purchase (recent versions) or $59.99/year subscription.

## Kopia: Modern and Fast

Kopia is a newer open-source backup tool with modern architecture, fast performance, and excellent deduplication.

### Installation

```bash
# macOS
brew install kopia

# Linux
sudo apt install kopia

# Windows
choco install kopia
```

### Basic Usage

```bash
# Create repository
kopia repository create filesystem --path /mnt/backup-repo

# Connect to existing repository
kopia repository connect filesystem --path /mnt/backup-repo

# Create snapshot
kopia snapshot create ~/Documents

# List snapshots
kopia snapshot list

# Restore files
kopia show <snapshot-id>
kopia restore <snapshot-id> ~/restore-location
```

### S3 Backend

```bash
# Create repository on AWS S3
kopia repository create s3 \
  --bucket=my-backups \
  --endpoint=s3.amazonaws.com

# Backup
kopia snapshot create ~/Documents

# Restore
kopia restore <snapshot-id> ~/restore
```

### Automation Script

```bash
#!/bin/bash
# Daily backup with Kopia

kopia repository connect filesystem --path /mnt/backups

# Backup multiple locations
kopia snapshot create ~/Documents
kopia snapshot create ~/Pictures
kopia snapshot create ~/Code

# Keep last 30 daily snapshots
kopia maintenance run --full

# Verify repository integrity
kopia repository verify
```

Schedule with cron:

```bash
0 2 * * * /home/user/kopia-backup.sh
```

### Strengths

- Modern architecture (written in Go)
- Very fast backup and restore
- Excellent deduplication
- Works with S3, Azure, Google Cloud
- Web UI available for easier management
- Free and open-source
- Active development

### Limitations

- Newer tool (smaller community than Borg/Restic)
- Repository format still evolving
- Documentation could be more

### Pricing

Free and open-source. Cloud storage costs depend on backend.

## Comparison Matrix

| Tool | Type | Dedup | Ease of Use | Cloud Support | Speed | Cost |
|------|------|-------|------------|---------------|-------|------|
| Borg | CLI | Excellent | Medium | SSH only | Fast | Free |
| Restic | CLI | Excellent | Easy | Multiple | Medium | Free |
| Duplicati | GUI | Fair | Very easy | Multiple | Slow | Free |
| Arq | App | Excellent | Very easy | Multiple | Fast | $69.99 |
| Kopia | CLI/Web | Excellent | Medium | Multiple | Very fast | Free |

## Backup Strategy by Use Case

**Individual home computer (100GB)**: Use Duplicati or Arq. Set and forget approach.

**Linux server (1TB+)**: Use Borg or Restic. Efficient storage, SSH access available.

**Team data (multi-device)**: Use Kopia with S3 backend. Fast performance, scales well.

**Budget-conscious**: Use Restic to AWS or Wasabi. Cheap cloud storage ($0.006/GB/month at Wasabi).

**Mac user wanting simplicity**: Use Arq. Best experience, one-time purchase.

## Restore Testing Checklist

Never skip restore testing. Backups don't exist if you can't restore:

- [ ] Test restoring full directory
- [ ] Test restoring single file from old backup
- [ ] Test restore to different location
- [ ] Verify restored file integrity (checksums)
- [ ] Test restore across version (e.g., restore Borg backup to different Borg version)
- [ ] Test disaster scenario (machine dies, restore from scratch)

Test each backup solution at least monthly.

## Encryption Key Safety

Your encryption key is critical. If lost, backups are unrecoverable:

- [ ] Store passphrase in password manager
- [ ] Keep physical backup of key (printed, in safe)
- [ ] Don't email passphrases
- [ ] Use strong passphrases (20+ characters)
- [ ] Never commit passphrases to git or config files

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
