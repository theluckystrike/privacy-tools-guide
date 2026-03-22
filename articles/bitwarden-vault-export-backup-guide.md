---
layout: default
title: "Bitwarden Vault Export Backup Guide"
description: "A practical guide to exporting and backing up your Bitwarden vault. Learn CLI methods, automation scripts, and recovery strategies for developers"
date: 2026-03-15
last_modified_at: 2026-03-22
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

To back up your Bitwarden vault, run `bw export --format json --encrypted` from the Bitwarden CLI to create a master-password-protected export of all your passwords, notes, and identities. For automated daily backups, wrap this command in a cron job that also handles session management and old backup cleanup. This guide covers all three export formats (JSON, encrypted JSON, CSV), step-by-step CLI setup, automated backup scripts, restore procedures, and security best practices for storing your exports.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Bitwarden Export Options

Bitwarden offers three primary export formats: JSON (unencrypted), CSV (unencrypted), and encrypted JSON. The format you choose depends on your threat model and recovery requirements.

The JSON export contains your complete vault structure including folder assignments, custom fields, and login URIs. CSV exports flatten this data into spreadsheet-compatible format but lose folder hierarchy and custom field information. The encrypted JSON option wraps your data in your master password-derived key, making it safe for cold storage.

One important caveat: none of the standard export formats includes file attachments. If you store documents, SSH keys, or other binary files as Bitwarden attachments, you must handle those separately. The CLI's `bw get attachment` command retrieves individual attachments, and you can script a loop over all items to pull them down before running your main export.

### Step 2: Exporting via Bitwarden CLI

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

### Step 3: Exporting via Web Interface

If you prefer the web vault, export from Settings > Export Vault:

1. Navigate to the web vault at vault.bitwarden.com
2. Go to Settings > Export Vault
3. Select your format (JSON or CSV)
4. Enter your master password to confirm
5. Download the exported file

The web interface offers the same three formats but requires manual download each time. For automated backups, use the CLI approach.

### Step 4: Exporting Organization Vaults

If you manage a Bitwarden Organizations account, the personal vault export does not include shared items stored in organization collections. Export organization vaults separately with admin privileges:

```bash
# List organization IDs
bw list organizations

# Export a specific organization vault
bw export --organizationid YOUR_ORG_ID --format json --output org-backup.json
```

Only organization Owners and Admins can export the organization vault. Members and Managers cannot, which is by design for access control. If you run automated org backups, use a dedicated service account with the minimum required role and store its credentials in a secrets manager like HashiCorp Vault or AWS Secrets Manager rather than in the cron script itself.

### Step 5: Automated Backup Scripts

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

### Verifying Backup Integrity

An unverified backup is not a backup. Add a checksum step to your script to detect silent corruption:

```bash
# After export, compute and store a SHA-256 checksum
sha256sum "$BACKUP_DIR/vault-$DATE.enc.json" > "$BACKUP_DIR/vault-$DATE.sha256"

# Later, verify with:
sha256sum --check "$BACKUP_DIR/vault-$DATE.sha256"
```

Pair this with a monitoring alert (via a cron status mailer or a tool like Healthchecks.io) so you get notified if the backup job fails silently.

### Step 6: Restore from Backup

When disaster strikes, Bitwarden CLI can restore your exported data:

```bash
# Import JSON backup
bw import --format json --vault ./vault-backup.json

# Or import CSV
bw import --format csv --vault ./vault-backup.csv
```

For encrypted backups, the import process automatically prompts for your master password to decrypt the file.

### Testing Restores

Do not wait for a disaster to discover your backup is broken. Test the restore process on a fresh Bitwarden account (you can use a free account on a different email address):

```bash
# 1. Log in to the test account
bw login test-restore@example.com

# 2. Import your encrypted backup
export BW_SESSION=$(bw unlock --raw)
bw import --format json --vault ~/path/to/vault-backup-encrypted.json

# 3. Verify item count matches your production vault
bw list items | jq length
```

Run this test every quarter. Set a calendar reminder so it does not slip.

## Security Considerations

Your vault backup contains sensitive data. Follow these practices:

Never store unencrypted JSON exports on cloud services. Use the `--encrypted` flag or encrypt the output with GPG:

```bash
gpg --symmetric --cipher-algo AES256 vault-backup.json
```

Keep at least one backup copy on offline media like an encrypted USB drive or secure paper backup of your encrypted export.

New passwords added since your last backup won't exist in older exports. Weekly automated exports ensure minimal data loss.

Periodically verify your backups work by restoring to a fresh vault. This catches format issues before you need the backup.

### Off-Site Storage Options

For off-site redundancy without trusting a cloud provider with plaintext data, use `rclone` to push encrypted exports to a provider of your choice:

```bash
# Configure rclone with your provider first (rclone config)
rclone copy "$BACKUP_DIR/vault-$DATE.enc.json" remote:bitwarden-backups/

# Or use rclone's built-in encryption layer (crypt remote) for an additional encryption layer
rclone copy "$BACKUP_DIR/vault-$DATE.enc.json" encrypted-remote:bitwarden-backups/
```

Supported providers include Backblaze B2, Wasabi, S3-compatible storage, and self-hosted solutions like MinIO. Since the file is already encrypted with your master password, the rclone crypt layer is optional but adds defense in depth.

### Step 7: Exporting Specific Items

For large vaults, export just what you need using filters:

```bash
# Export items from a specific folder
bw list items --folderid folder-id | jq '.' > folder-backup.json

# Export favorites only
bw list items --favorite true > favorites-backup.json
```

These targeted exports help when migrating specific services or creating limited backup sets.

### Step 8: Bitwarden Send for Sensitive Sharing

For sharing individual credentials securely, Bitwarden Send complements your backup strategy:

```bash
# Create a send with auto-delete after 1 view
bw send --file ~/secret-document.txt --max-access-count 1
```

This generates a link that works once, then disappears—useful for sharing recovery keys or emergency credentials with trusted contacts.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to complete this setup?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)
- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [Bitwarden Web Vault vs Desktop App Comparison](/privacy-tools-guide/bitwarden-web-vault-vs-desktop-app-comparison/)
- [Set Up Bitwarden Emergency Access for Password Vault](/privacy-tools-guide/how-to-set-up-bitwarden-emergency-access-for-password-vault-/)
- [Migrating From Keepass Database To Bitwarden Cloud Vault](/privacy-tools-guide/migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
