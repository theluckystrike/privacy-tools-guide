---
layout: default
title: "Migrating From NordPass to Bitwarden"
description: "A technical guide for developers and power users migrating passwords from NordPass to Bitwarden. Covers CLI export, import automation, custom fields"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /migrating-from-nordpass-to-bitwarden-export-import-process-guide/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---
---
layout: default
title: "Migrating From NordPass to Bitwarden"
description: "A technical guide for developers and power users migrating passwords from NordPass to Bitwarden. Covers CLI export, import automation, custom fields"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /migrating-from-nordpass-to-bitwarden-export-import-process-guide/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true---

{% raw %}

Migrate from NordPass to Bitwarden by exporting to CSV through the desktop app or CLI, then importing into Bitwarden using the web interface or bitwarden CLI tool. Both methods preserve passwords, folders, and custom fields, though some NordPass-specific features require manual remapping.

## Understanding Export Formats

NordPass and Bitwarden use different export mechanisms. NordPass provides encrypted exports that require specific handling, while Bitwarden accepts multiple import formats including CSV and JSON.

Before starting the migration, ensure you have:
- Your NordPass master password accessible
- A Bitwarden account created (free tier works for imports)
- The Bitwarden CLI installed for programmatic handling

## Exporting Data From NordPass

NordPass offers several export methods. The most reliable approach for full data transfer uses their desktop application or CLI.

### Desktop Application Export

1. Open NordPass and log in with your master password
2. Navigate to Settings → Export
3. Select the export format (CSV recommended for maximum compatibility)
4. Enter your master password to confirm
5. Save the exported file to a secure location

The CSV export includes most standard fields: site URLs, usernames, passwords, notes, and folder assignments.

### CLI Export (Recommended for Automation)

For developers preferring command-line workflows, install the NordPass CLI:

```bash
# Install NordPass CLI via npm
npm install -g nordpass-cli

# Authenticate with your Nord account
np login

# Export vault to CSV
np export --format csv --output ~/nordpass-export.csv

# Verify the export file
head -5 ~/nordpass-export.csv
```

The CLI export provides the same data as the desktop application but enables scripting for large vaults.

### Handling Encrypted Exports

NordPass also offers an encrypted backup that preserves more data types:

```bash
# Create encrypted backup (includes all items)
np backup create --output ~/nordpass-backup.npe

# This file requires NordPass to decrypt—use for full restoration only
```

For migration purposes, the CSV format provides the best balance of compatibility and field coverage.

## Importing Into Bitwarden

Bitwarden supports multiple import formats. The CSV format from NordPass maps cleanly to Bitwarden's import structure.

### Direct CSV Import via Web Vault

1. Log into vault.bitwarden.com
2. Navigate to Tools → Import
3. Select "NordPass (CSV)" as the source
4. Upload your exported CSV file
5. Review the import preview and confirm

### CLI Import (Programmatic Approach)

The Bitwarden CLI provides more control over the import process:

```bash
# Install Bitwarden CLI
# macOS: brew install bitwarden-cli
# Linux: sudo apt install bitwarden

# Log in to Bitwarden
bw login your@email.com

# Unlock vault and copy session key
export BW_SESSION=$(bw unlock --raw)

# Import the NordPass CSV
bw import bitwardencsv ~/nordpass-export.csv --formats

# Alternatively, use generic CSV format
bw import csv ~/nordpass-export.csv

# Verify import count
echo "Import completed. Check vault for $(bw list items | jq 'length') items"
```

### Handling Custom Fields

NordPass supports custom fields that require special attention during migration. The CSV export includes custom fields in a specific format that Bitwarden can parse:

```csv
url,username,password,note,custom_fields
github.com,dev@example.com,super_secret_pass,"API key notes",{"api_key":"value123"}
```

If custom fields don't import correctly, use the Bitwarden CLI to add them programmatically:

```bash
# Get the item ID for a specific login
ITEM_ID=$(bw get item "github.com" --pretty | jq -r '.id')

# Add a custom field to the item
bw get item $ITEM_ID | jq '. |= .+ {"fields": [{"name": "api_key", "value": "value123", "type": 0}]}' | bw edit item $ITEM_ID
```

## Migrating TOTP Authenticator Data

Both NordPass and Bitwarden can store TOTP (time-based one-time password) secrets. This requires manual intervention since CSV exports don't reliably transfer TOTP keys.

### Export TOTP Secrets From NordPass

NordPass stores TOTP secrets in the vault. To export:

1. Open NordPass desktop app
2. Select each item with TOTP enabled
3. Click the TOTP field to reveal the secret
4. Copy the secret (format: `otpauth://totp/...` or base32 key)

### Import TOTP Into Bitwarden

Bitwarden allows TOTP key import via the web vault:

1. Edit the imported login item
2. Navigate to "Advanced" section
3. Find "TOTP" field
4. Paste the TOTP secret (base32 format)
5. Save the item

For bulk TOTP migration, use this CLI approach:

```bash
# Extract TOTP secrets from NordPass export (manual process)
# Format expected: base32 secret key

# Add TOTP to multiple items via script
while read line; do
  ITEM_NAME=$(echo $line | cut -d',' -f1)
  TOTP_SECRET=$(echo $line | cut -d',' -f6)

  ITEM_ID=$(bw list items --search "$ITEM_NAME" | jq -r '.[0].id')

  if [ "$ITEM_ID" != "null" ] && [ -n "$TOTP_SECRET" ]; then
    bw get item $ITEM_ID | jq --arg secret "$TOTP_SECRET" '.totp = $secret' | bw edit item $ITEM_ID
    echo "Added TOTP to: $ITEM_NAME"
  fi
done < totp-secrets.csv
```

## Preserving Folder Organization

NordPass uses folders and tags. Bitwarden uses collections and folders. The CSV import typically creates Bitwarden folders automatically:

```csv
folder,url,username,password,note
Work,github.com,work@example.com,pass123,"Work account"
Personal,netflix.com,personal@example.com,pass456,"Personal account"
```

After import, verify folder structure:

```bash
# List all folders in Bitwarden
bw list folders

# Create new folders if needed
bw create folder "Development"
```

## Automation Script for Complete Migration

Here's a bash script that automates the entire process:

```bash
#!/bin/bash
# migrate-nordpass-to-bitwarden.sh

set -e

# Configuration
EXPORT_FILE="~/nordpass-export.csv"
BITWARDEN_EMAIL="your@email.com"

echo "Starting NordPass to Bitwarden migration..."

# Step 1: Export from NordPass (assumes CLI is installed)
echo "[1/5] Exporting from NordPass..."
np export --format csv --output $EXPORT_FILE

# Step 2: Log into Bitwarden
echo "[2/5] Logging into Bitwarden..."
bw login $BITWARDEN_EMAIL
export BW_SESSION=$(bw unlock --raw)

# Step 3: Import data
echo "[3/5] Importing vault data..."
bw import csv $EXPORT_FILE

# Step 4: Sync to ensure data is on server
echo "[4/5] Syncing vault..."
bw sync

# Step 5: Verify import
echo "[5/5] Verifying import..."
ITEM_COUNT=$(bw list items | jq 'length')
echo "Migration complete. Vault contains $ITEM_COUNT items."

# Cleanup
unset BW_SESSION
echo "Done. Remember to secure-delete the export file."
```

## Post-Migration Verification

After importing, verify your data integrity:

```bash
# List all imported items
bw list items | jq '.[] | {name: .name, url: .login.uri, username: .login.username}'

# Check for any items without passwords (potential import issues)
bw list items | jq '.[] | select(.login.password == null)'

# Verify TOTP functionality
# Each item should show a TOTP code in the web vault
```

Common verification steps:
1. Confirm all passwords are readable (no import errors)
2. Test login autofill for a few critical accounts
3. Verify TOTP codes sync correctly for accounts with 2FA
4. Check that folders/collections preserved correctly
5. Delete the NordPass export file securely

## Security Considerations

During migration, handle your data carefully:

- **Temporary files**: Delete the CSV export after successful import
- **Clipboard**: Clear clipboard after copying passwords
- **Session tokens**: CLI session tokens expire; re-authenticate as needed
- **Master password**: Never share or store your master password in scripts

For enhanced security during migration, consider running these commands on an offline machine:

```bash
# Securely delete export file after import
shred -u ~/nordpass-export.csv

# Clear bash history containing any sensitive commands
history -c
```

## Troubleshooting Common Issues

**Issue**: CSV import fails with encoding errors

**Solution**: Ensure the CSV uses UTF-8 encoding:
```bash
iconv -f ISO-8859-1 -t UTF-8 nordpass-export.csv > nordpass-utf8.csv
```

**Issue**: Custom fields not appearing

**Solution**: Manually add via CLI or web interface, as CSV custom field support varies

**Issue**: Duplicate entries after import

**Solution**: Use Bitwarden's deduplication feature or import to a fresh vault first, then merge

**Issue**: TOTP codes not working

**Solution**: Verify the secret key format—Bitwarden requires base32, not otpauth:// URIs

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does Bitwarden offer a free tier?**

Most major tools offer some form of free tier or trial period. Check Bitwarden's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [ProtonMail Import Export Tool Guide](/privacy-tools-guide/protonmail-import-export-tool-guide/)
- [Bitwarden vs NordPass Comparison 2026](/privacy-tools-guide/bitwarden-vs-nordpass-comparison-2026/)
- [Bitwarden Vault Export Backup Guide](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [Migrating From Icloud Keychain To Bitwarden Complete Transfe](/privacy-tools-guide/migrating-from-icloud-keychain-to-bitwarden-complete-transfe/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
