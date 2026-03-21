---
layout: default
title: "Encrypt Cloud Storage with Rclone Before Uploading"
description: "Use rclone crypt to encrypt files client-side before syncing to Dropbox, Google Drive, or S3 so the provider never has access to your plaintext data"
date: 2026-03-21
author: theluckystrike
permalink: /secure-cloud-storage-encryption-rclone/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

Dropbox, Google Drive, and S3 encrypt your data in transit and at rest — but they hold the encryption keys. This means they can read your files, hand them to law enforcement, scan them for content policy violations, and potentially expose them in a breach. Rclone's built-in encryption layer encrypts your files on your machine before they ever leave it, using keys that only you hold.

This guide configures rclone with client-side encryption on top of any cloud storage provider.

## How rclone crypt Works

rclone crypt is a transparent encryption layer that wraps any rclone remote. When you sync to a crypt remote:

1. Your local files are encrypted using XSalsa20 (stream cipher) + Poly1305 (MAC)
2. File and directory names are also encrypted (optional but recommended)
3. The encrypted data is uploaded to your cloud provider
4. When syncing back, rclone decrypts on the fly

The cloud provider sees only encrypted blobs with encrypted names. Even if they have full access to your account, the data is unreadable without your passphrase.

## Installation

### Linux

```bash
# Via package manager
sudo apt install rclone   # Debian/Ubuntu
sudo dnf install rclone   # Fedora

# Or install the latest version directly
curl https://rclone.org/install.sh | sudo bash

# Verify
rclone --version
```

### macOS

```bash
brew install rclone
```

### Windows

Download the installer from rclone.org/downloads/

## Step 1: Configure Your Cloud Provider Remote

First set up the base remote (the actual cloud storage). Rclone has interactive setup:

```bash
rclone config
```

Select `n` for new remote, choose your provider (e.g., `drive` for Google Drive, `dropbox`, `s3` for AWS S3, `b2` for Backblaze B2), and follow the authentication prompts.

After authentication, test the connection:

```bash
rclone ls gdrive:                    # List files on Google Drive
rclone about gdrive:                 # Show storage quota
```

## Step 2: Create an Encrypted Remote

Add a crypt remote layered on top of your cloud remote:

```bash
rclone config
```

1. Select `n` for new remote
2. Name it (e.g., `gdrive-crypt`)
3. Type: `crypt`
4. Remote: `gdrive:encrypted-folder` (point to a subfolder on the cloud remote)
5. Filename encryption: `standard` (encrypts and obfuscates names; use `obfuscate` if you want a weaker form that's easier to debug)
6. Directory name encryption: `true`
7. Enter and confirm a password (this is your master encryption key — store it safely)
8. Optionally add a salt password (additional entropy; must be the same when decrypting)

The configuration is stored in `~/.config/rclone/rclone.conf` — protect this file:

```bash
chmod 600 ~/.config/rclone/rclone.conf
```

## Step 3: Sync and Decrypt

All rclone commands work identically with the crypt remote:

```bash
# Copy local files to encrypted cloud storage
rclone copy /local/documents gdrive-crypt:documents/

# Sync (mirror) — deletes destination files not in source
rclone sync /local/documents gdrive-crypt:documents/

# Copy from encrypted cloud to local (auto-decrypts)
rclone copy gdrive-crypt:documents/ /local/restore/

# List files (shows decrypted names)
rclone ls gdrive-crypt:

# Check for differences
rclone check /local/documents gdrive-crypt:documents/
```

What the cloud provider sees on their end:

```
# On Google Drive (encrypted, unreadable)
encrypted-folder/
  a6vtu8kpqxlzz4hn5mw3jy0b7d2i9e1f/       ← encrypted directory name
    8xn2p4kytw5ql6mr3vj0ia9dc7ue1hfb       ← encrypted filename
    k3bz7q9wnl1pt8vs2rm0oy4dxi6juf5h       ← encrypted filename
```

What you see with rclone:

```
documents/
  financial-records-2026.pdf
  contract-draft.docx
```

## Step 4: Automate Backups

Create a backup script:

```bash
#!/bin/bash
# /usr/local/bin/backup-encrypted
# Daily encrypted backup to cloud storage

set -euo pipefail

LOCAL_DIR="/home/$USER/documents"
REMOTE_DIR="gdrive-crypt:backup/documents"
LOG_FILE="/var/log/rclone-backup.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] Starting backup of $LOCAL_DIR" >> "$LOG_FILE"

rclone sync "$LOCAL_DIR" "$REMOTE_DIR" \
  --checksum \
  --transfers 4 \
  --log-file "$LOG_FILE" \
  --log-level INFO \
  --exclude "*.tmp" \
  --exclude ".cache/**"

echo "[$TIMESTAMP] Backup completed" >> "$LOG_FILE"
```

Schedule with cron:

```bash
chmod +x /usr/local/bin/backup-encrypted
crontab -e
# Add: 0 2 * * * /usr/local/bin/backup-encrypted
```

## Step 5: Mount Encrypted Storage as a Local Filesystem

rclone can mount your encrypted cloud storage as a local directory, making it appear as a normal folder:

```bash
# Create mount point
mkdir -p ~/cloud-documents

# Mount (runs in foreground; use --daemon for background)
rclone mount gdrive-crypt:documents/ ~/cloud-documents \
  --vfs-cache-mode writes \
  --vfs-cache-max-age 24h \
  --daemon

# Now ~/cloud-documents shows decrypted files transparently
ls ~/cloud-documents

# Unmount
fusermount -u ~/cloud-documents   # Linux
# or
umount ~/cloud-documents          # macOS
```

For automatic mounting on login, add to your systemd user service:

```bash
# ~/.config/systemd/user/rclone-gdrive.service
[Unit]
Description=rclone gdrive mount
After=network-online.target

[Service]
ExecStart=/usr/bin/rclone mount gdrive-crypt:documents/ %h/cloud-documents \
  --vfs-cache-mode writes \
  --vfs-cache-max-age 24h \
  --log-level ERROR
ExecStop=/bin/fusermount -u %h/cloud-documents
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable rclone-gdrive
systemctl --user start rclone-gdrive
```

## Step 6: Back Up the Encryption Password

Your rclone crypt configuration and password are the only way to decrypt your data. If you lose both, the data is unrecoverable.

```bash
# Show your rclone config (contains the encrypted password)
rclone config show

# Export the crypt section for backup
grep -A10 '\[gdrive-crypt\]' ~/.config/rclone/rclone.conf
```

Store this configuration and your plaintext password in:
- Your password manager (under a dedicated entry)
- An encrypted local backup (on a separate drive from the cloud backup)
- Printed copy in a physically secure location

## Encryption Strength

rclone crypt uses:
- **XSalsa20** stream cipher for file content
- **Poly1305** MAC for authentication
- **scrypt** key derivation from your passphrase
- **PKCS7** padding to prevent block size leaks
- **AES-256** in CTR mode for file name encryption

This provides authenticated encryption — any tampering with the ciphertext is detectable.

The passphrase and optional salt together generate the master encryption key. A strong Diceware passphrase of 6+ words provides sufficient security against offline brute force.

## Related Reading

- [Diceware Passphrase Generation Guide](/diceware-passphrase-generation-guide/)
- [Nextcloud End-to-End Encryption Setup Guide](/nextcloud-end-to-end-encryption-setup-guide/)
- [Secure File Deletion on SSD Drives](/secure-file-deletion-ssd-drives-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
