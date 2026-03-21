---
layout: default
title: "Restic Encrypted Backup Setup Guide"
description: "Set up automated encrypted backups with restic. Covers local, S3, and Backblaze B2 backends, snapshot policies, and scheduled automation."
date: 2026-03-21
author: theluckystrike
permalink: restic-encrypted-backup-setup-guide
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Restic encrypts every backup with AES-256-CTR before it leaves your machine. Even if your backup storage is compromised, the attacker gets ciphertext. This guide covers installation, initializing repositories on local storage, S3-compatible buckets, and Backblaze B2, plus automating backups with systemd or cron.

## Install Restic

```bash
# Debian/Ubuntu
sudo apt install restic

# Fedora/RHEL
sudo dnf install restic

# macOS
brew install restic

# Or download a binary directly
curl -L https://github.com/restic/restic/releases/latest/download/restic_linux_amd64.bz2 | bunzip2 > restic
chmod +x restic
sudo mv restic /usr/local/bin/
```

Verify:

```bash
restic version
```

## Concepts

- **Repository**: A storage location (directory, S3 bucket, SFTP server) that holds encrypted backup data.
- **Snapshot**: A point-in-time backup of a set of files.
- **Password**: The key that encrypts the entire repository. Losing it means losing all your backups.

Store your repository password in a password manager. Do not rely on memory alone.

## Option 1: Local Repository

Initialize a repository on a local drive or external disk:

```bash
restic init --repo /mnt/backup/restic-repo
```

Restic asks for a password twice. Store this password securely.

Back up your home directory:

```bash
restic -r /mnt/backup/restic-repo backup ~/
```

The first backup transfers everything. Subsequent backups are incremental — only changed blocks are uploaded.

## Option 2: S3-Compatible Storage (Wasabi, MinIO, etc.)

Set environment variables for credentials:

```bash
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export RESTIC_REPOSITORY=s3:https://s3.wasabisys.com/your-bucket-name
export RESTIC_PASSWORD=your-strong-password
```

Initialize the repository:

```bash
restic init
```

Run a backup:

```bash
restic backup ~/Documents ~/Projects
```

For AWS S3, use:

```bash
export RESTIC_REPOSITORY=s3:s3.amazonaws.com/your-bucket-name
```

## Option 3: Backblaze B2

Backblaze B2 has native restic support and is inexpensive (~$0.006/GB/month):

```bash
export B2_ACCOUNT_ID=your-account-id
export B2_ACCOUNT_KEY=your-application-key
export RESTIC_REPOSITORY=b2:your-bucket-name:/restic
export RESTIC_PASSWORD=your-strong-password
```

Initialize and back up:

```bash
restic init
restic backup ~/Documents
```

## Using a Password File

Typing the password every time is tedious for automation. Store it in a file with restricted permissions:

```bash
echo "your-strong-password" > ~/.restic-password
chmod 600 ~/.restic-password
```

Use it with every command:

```bash
restic -r /mnt/backup/restic-repo --password-file ~/.restic-password backup ~/Documents
```

Or set an environment variable:

```bash
export RESTIC_PASSWORD_FILE=~/.restic-password
```

## Exclude Files and Directories

Create an exclusions file:

```bash
nano ~/.restic-excludes
```

```
/home/user/.cache
/home/user/.local/share/Trash
/home/user/Downloads
*.tmp
*.log
node_modules/
.git/
```

Run backup with exclusions:

```bash
restic -r /mnt/backup/restic-repo backup ~/ --exclude-file=~/.restic-excludes
```

## List and Browse Snapshots

List all snapshots:

```bash
restic -r /mnt/backup/restic-repo snapshots
```

Output:
```
ID        Time                 Host        Tags        Paths
-----------------------------------------------------------------------
3d8f2a1b  2026-03-21 02:00:00  myhost                  /home/user
c7e9f4d2  2026-03-20 02:00:00  myhost                  /home/user
```

Browse files inside a snapshot:

```bash
restic -r /mnt/backup/restic-repo ls 3d8f2a1b
```

Mount a snapshot as a filesystem:

```bash
mkdir /tmp/restic-mount
restic -r /mnt/backup/restic-repo mount /tmp/restic-mount
ls /tmp/restic-mount/snapshots/latest/
```

## Restore Files

Restore a single file or directory from the latest snapshot:

```bash
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore --include /home/user/Documents/important.pdf
```

Restore an entire snapshot:

```bash
restic -r /mnt/backup/restic-repo restore latest --target /tmp/full-restore
```

Restore to the original path (overwrites existing files):

```bash
restic -r /mnt/backup/restic-repo restore latest --target /
```

## Snapshot Retention Policy

Keeping every snapshot forever wastes storage. Set a retention policy:

```bash
restic -r /mnt/backup/restic-repo forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --prune
```

This keeps:
- The last 7 daily snapshots
- 4 weekly snapshots
- 6 monthly snapshots
- Removes data no longer referenced by any kept snapshot (`--prune`)

## Automate with systemd

Create a systemd service:

```bash
sudo nano /etc/systemd/system/restic-backup.service
```

```ini
[Unit]
Description=Restic Backup
After=network.target

[Service]
Type=oneshot
User=youruser
Environment=RESTIC_REPOSITORY=/mnt/backup/restic-repo
Environment=RESTIC_PASSWORD_FILE=/home/youruser/.restic-password
ExecStart=/usr/bin/restic backup /home/youruser --exclude-file=/home/youruser/.restic-excludes
ExecStartPost=/usr/bin/restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

Create a timer:

```bash
sudo nano /etc/systemd/system/restic-backup.timer
```

```ini
[Unit]
Description=Run Restic Backup Daily

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now restic-backup.timer
```

Check timer status:

```bash
systemctl list-timers restic-backup.timer
```

## Verify Repository Integrity

Run this regularly to detect data corruption:

```bash
restic -r /mnt/backup/restic-repo check
```

For a full data verification (reads all pack files):

```bash
restic -r /mnt/backup/restic-repo check --read-data
```

This is slow on large repos but confirms backup integrity end to end.

## Key Rotation

To change the repository password:

```bash
restic -r /mnt/backup/restic-repo key passwd
```

List all keys associated with the repo:

```bash
restic -r /mnt/backup/restic-repo key list
```

Remove an old key by its ID:

```bash
restic -r /mnt/backup/restic-repo key remove <key-id>
```

## Related Reading

- [Restic vs Borg Backup Encrypted Comparison](/restic-vs-borg-backup-encrypted-comparison)
- [Best Encrypted Backup Solution for Developers](/best-encrypted-backup-solution-for-developers)
- [Best Encrypted Cloud Storage 2026](/best-encrypted-cloud-storage-2026)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
