---
layout: default
title: "Bitwarden vs LastPass Migration Guide"
description: "A practical guide for developers migrating from LastPass to Bitwarden. Covers CLI-based export/import, handling TOTP codes, custom fields, and."
date: 2026-03-15
author: theluckystrike
permalink: /bitwarden-vs-lastpass-migration-guide/
categories: [comparisons]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, comparison]
---

{% raw %}

# Bitwarden vs LastPass Migration Guide: Complete Technical Walkthrough

Migrating password managers requires careful handling of sensitive data. For developers and power users, the transition from LastPass to Bitwarden offers improved open-source transparency, self-hosting options, and better CLI tooling. This guide walks through the technical process using command-line tools for maximum control and automation.

## Why Developers Choose Bitwarden Over LastPass

Several technical factors make Bitwarden the preferred choice for developers:

- **Open-source codebase**: Audit the encryption, sync, and storage implementations yourself
- **Self-hosting option**: Run your own vault server using Docker
- **CLI-first workflow**: Bitwarden's CLI is purpose-built for scripting and CI/CD integration
- **No team size limits**: Free organizations support unlimited users
- **Better pricing**: Premium features cost $10/year versus LastPass's $36/year

## Preparing for Migration

Before starting, ensure you have:

1. A Bitwarden account (free or premium)
2. LastPass credentials with admin access to your vault
3. Node.js installed (for Bitwarden CLI)
4. jq or similar JSON processing tools

```bash
# Install Bitwarden CLI
npm install -g @bitwarden/cli

# Verify installation
bw --version
```

## Exporting Data from LastPass

LastPass provides CSV export through their browser extension or CLI. For programmatic control, use the LastPass CLI:

```bash
# Install LastPass CLI (Linux/macOS)
brew install lastpass-cli  # macOS
# or
sudo apt install lpass    # Debian/Ubuntu

# Export vault to CSV
lpass export --format=csv > lastpass-export.csv
```

The CSV export includes:
- URL
- Username
- Password
- Extra (notes)
- Name (folder/title)
- Favicon (ignored by Bitwarden)

### Handling TOTP Seeds

LastPass exports TOTP codes as "One-Time Passwords" in the Extra field. You'll need to extract these manually for Bitwarden's TOTP support:

```bash
# Extract TOTP seeds using grep and sed
# TOTP secrets typically appear as base32 strings
grep -oP '(?<=TOTP: )[A-Z2-7]+=*' lastpass-export.csv > totp-seeds.txt
```

## Importing into Bitwarden

Bitwarden's CLI handles imports directly:

```bash
# Login to Bitwarden
bw login your@email.com

# Unlock vault (will prompt for master password)
bw unlock

# Import the LastPass CSV
# --format lastpasscsv tells Bitwarden how to parse the data
bw import --format lastpasscsv --file ./lastpass-export.csv
```

The import process maps LastPass fields to Bitwarden equivalents:
- URL → URI
- Username → Username
- Password → Password
- Extra → Notes
- Name → Folder/Name

## Migrating TOTP Authenticators

LastPass stores TOTP secrets in a separate format. For full migration:

### Method 1: Manual Export via Browser Extension

1. Open LastPass browser extension
2. Navigate to Account Settings → Export
3. Export "Export Generic CSV"
4. Extract the "One-Time Password" column

### Method 2: Using a Python Script

```python
#!/usr/bin/env python3
import csv
import sys

def parse_totp_from_csv(input_file, output_file):
    """Extract TOTP secrets from LastPass export"""
    totp_entries = []
    
    with open(input_file, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        for row in reader:
            extra = row.get('Extra', '')
            # Look for TOTP patterns in the notes
            if 'TOTP:' in extra or 'totp:' in extra.lower():
                # Parse and clean the secret
                for line in extra.split('\n'):
                    if 'TOTP' in line or 'totp' in line.lower():
                        parts = line.split(':')
                        if len(parts) >= 2:
                            secret = parts[-1].strip()
                            totp_entries.append({
                                'url': row['url'],
                                'name': row['name'],
                                'secret': secret
                            })
    
    # Output for manual TOTP entry
    with open(output_file, 'w') as f:
        for entry in totp_entries:
            f.write(f"{entry['name']}: {entry['secret']}\n")
    
    print(f"Found {len(totp_entries)} TOTP entries")
    return totp_entries

if __name__ == '__main__':
    parse_totp_from_csv('lastpass-export.csv', 'totp-seeds.txt')
```

### Adding TOTP to Bitwarden

After migration, manually add TOTP to each login item:

```bash
# Get item ID
bw list items --url "github.com"

# Update with TOTP (requires premium for TOTP)
bw edit item <item-id> --totp "JBSWY3DPEHPK3PXP"
```

## Bulk Operations with Bitwarden CLI

For large vaults, use scripting for efficiency:

```bash
#!/bin/bash
# Sync vault after import
bw sync

# List all items with specific tags
bw list items --search "" | jq '.[] | select(.organizationId == null)'

# Export Bitwarden vault for backup
bw export --format json --output ./bitwarden-backup.json
```

## Folder Organization

LastPass uses folders; Bitwarden uses collections for organizations and folders for personal vaults. Import automatically creates folders:

```bash
# View imported folders
bw list folders

# Reorganize using CLI
bw move <item-id> <folder-name>
```

## Verification Checklist

After migration, verify your data:

- [ ] All passwords imported with correct values
- [ ] URLs match original entries
- [ ] Notes field contains all original data
- [ ] TOTP codes work (test with `bw get item <id>`)
- [ ] Folder structure matches LastPass

```bash
# Verify specific entry
bw get item "GitHub" | jq '{username, password, totp, notes}'
```

## Automating Continuous Migration

For team migrations, script the process:

```javascript
// migration-script.js
const { execSync } = require('child_process');

function migrateEntry(entry) {
  try {
    // Create item in Bitwarden
    const cmd = `bw create item --organization-id ${process.env.BW_ORG_ID}`;
    execSync(cmd, {
      input: JSON.stringify(entry),
      encoding: 'utf-8'
    });
    console.log(`Migrated: ${entry.name}`);
  } catch (err) {
    console.error(`Failed: ${entry.name}`, err.message);
  }
}
```

## Security Considerations

During migration:

1. **Use a clean environment**: Run on a secure machine without malware
2. **Encrypt exports**: Store temporary files encrypted
3. **Delete intermediates**: Remove CSV files after successful import
4. **Rotate passwords**: Consider updating critical passwords post-migration
5. **Enable 2FA**: Set up Bitwarden with TOTP or FIDO2

```bash
# Secure cleanup
shred -u lastpass-export.csv
shred -u totp-seeds.txt
```

## Related Reading

- [Best Hardware Security Key for Developers: A Practical Guide](/privacy-tools-guide/best-hardware-security-key-for-developers/)
- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [Telegram vs Signal: Which Is Actually Safer? A Technical.](/privacy-tools-guide/telegram-vs-signal-which-is-actually-safer/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
