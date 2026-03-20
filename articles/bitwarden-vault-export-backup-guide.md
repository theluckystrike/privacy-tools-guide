---


layout: default
title: "Bitwarden Vault Export Backup Guide: Complete Technical Reference"
description: "A practical guide to exporting and backing up your Bitwarden vault. Learn CLI methods, automation scripts, and recovery strategies for developers."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /bitwarden-vault-export-backup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---


{% raw %}
# Bitwarden Vault Export Backup Guide: Complete Technical Reference

To back up your Bitwarden vault, run `bw export --format json --encrypted` from the Bitwarden CLI to create a master-password-protected export of all your passwords, notes, and identities. For automated daily backups, wrap this command in a cron job that also handles session management and old backup cleanup. This guide covers all three export formats (JSON, encrypted JSON, CSV), step-by-step CLI setup, automated backup scripts, restore procedures, and security best practices for storing your exports.

## Understanding Bitwarden Export Options

Bitwarden offers three primary export formats: JSON (unencrypted), CSV (unencrypted), and encrypted JSON. The format you choose depends on your threat model and recovery requirements.

The JSON export contains your complete vault structure including folder assignments, custom fields, and login URIs. CSV exports flatten this data into spreadsheet-compatible format but lose folder hierarchy and custom field information. The encrypted JSON option wraps your data in your master password-derived key, making it safe for cold storage.

## Exporting via Bitwarden CLI

The Bitwarden CLI provides the most flexible export capabilities. Install it first if you haven't already:

```bash
# Install via npm
npm install -g @bitwarden/cli

# Or on macOS with Homebrew
brew install bitwarden-cli
```

Authenticate with your vault before exporting:

```bash
bw login your@email.com
# Enter your master password when prompted

# Unlock your vault and store the session key
export BW_SESSION=$(bw unlock --raw)
```

### JSON Export

The basic JSON export includes all vault items in readable format:

```bash
bw export --output ./vault-backup.json --format json
```

This command creates a file containing every login, note, card, and identity item in your vault. The structure looks like:

```json
{
  "folders": [
    { "id": "folder-id", "name": "Work Accounts" }
  ],
  "items": [
    {
      "id": "item-id",
      "folderId": "folder-id",
      "type": 1,
      "name": "GitHub",
      "login": {
        "username": "dev@example.com",
        "password": "actual-password-here",
        "totp": "otpauth://..."
      }
    }
  ]
}
```

### Encrypted JSON Export

For secure backups that don't expose plaintext passwords, use the encrypted format:

```bash
bw export --output ./vault-backup-encrypted.json --format json --encrypted
```

This export uses your master password to encrypt the output. To restore this backup, you'll need both the encrypted file and your master password.

### CSV Export

CSV format works well for migration to other password managers or spreadsheet analysis:

```bash
bw export --output ./vault-backup.csv --format csv
```

The CSV output includes folder, favorite, item name, username, password, URL, notes, and custom fields as columns. Be aware that multi-value custom fields get concatenated.

## Exporting via Web Interface

If you prefer the web vault, export from Settings > Export Vault:

1. Navigate to the web vault at vault.bitwarden.com
2. Go to Settings > Export Vault
3. Select your format (JSON or CSV)
4. Enter your master password to confirm
5. Download the exported file

The web interface offers the same three formats but requires manual download each time. For automated backups, use the CLI approach.

## Automated Backup Scripts

For reliable disaster recovery, automate your exports with cron jobs. Here's a practical bash script:

```bash
#!/bin/bash

# Bitwarden automated backup script
# Set these environment variables in your CI/CD or cron environment
BW_EMAIL="your@email.com"
BACKUP_DIR="/path/to/secure/backup/location"
DATE=$(date +%Y-%m-%d)

# Create encrypted backup
cd /tmp
echo "$BW_MASTER_PASSWORD" | bw login --passwordenv BW_MASTER_PASSWORD
export BW_SESSION=$(bw unlock --raw)

# Export encrypted JSON for secure storage
bw export --output "$BACKUP_DIR/vault-$DATE.enc.json" --format json --encrypted

# Also export CSV for compatibility
bw export --vault "$BACKUP_DIR/vault-$DATE.csv" --format csv

# Clean up session
bw lock

# Remove backups older than 30 days
find "$BACKUP_DIR" -name "vault-*.enc.json" -mtime +30 -delete
find "$BACKUP_DIR" -name "vault-*.csv" -mtime +30 -delete
```

Schedule this script in your crontab:

```bash
# Run daily at 2 AM
0 2 * * * /path/to/backup-script.sh >> /var/log/bitwarden-backup.log 2>&1
```

## Restore from Backup

When disaster strikes, Bitwarden CLI can restore your exported data:

```bash
# Import JSON backup
bw import --format json --vault ./vault-backup.json

# Or import CSV
bw import --format csv --vault ./vault-backup.csv
```

For encrypted backups, the import process automatically prompts for your master password to decrypt the file.

## Security Considerations

Your vault backup contains sensitive data. Follow these practices:

Never store unencrypted JSON exports on cloud services. Use the `--encrypted` flag or encrypt the output with GPG:

```bash
gpg --symmetric --cipher-algo AES256 vault-backup.json
```

Keep at least one backup copy on offline media like an encrypted USB drive or secure paper backup of your encrypted export.

New passwords added since your last backup won't exist in older exports. Weekly automated exports ensure minimal data loss.

Periodically verify your backups work by restoring to a fresh vault. This catches format issues before you need the backup.

## Exporting Specific Items

For large vaults, export just what you need using filters:

```bash
# Export items from a specific folder
bw list items --folderid folder-id | jq '.' > folder-backup.json

# Export favorites only
bw list items --favorite true > favorites-backup.json
```

These targeted exports help when migrating specific services or creating limited backup sets.

## Bitwarden Send for Sensitive Sharing

For sharing individual credentials securely, Bitwarden Send complements your backup strategy:

```bash
# Create a send with auto-delete after 1 view
bw send --file ~/secret-document.txt --max-access-count 1
```

This generates a link that works once, then disappears—useful for sharing recovery keys or emergency credentials with trusted contacts.

## Related Reading

- [Best Password Manager for Small Business: A Developer Guide](/privacy-tools-guide/best-password-manager-for-small-business/)
- [1Password Watchtower Feature Review: A Developer's Guide](/privacy-tools-guide/1password-watchtower-feature-review/)
- [Best Password Manager for Developers: A Practical Guide](/privacy-tools-guide/best-password-manager-for-developers/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
