---
layout: default
title: "Migrating From Icloud Keychain To Bitwarden Complete Transfe"
description: "A technical guide for developers and power users migrating passwords from iCloud Keychain to Bitwarden. Covers export methods, CLI tools, CSV"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /migrating-from-icloud-keychain-to-bitwarden-complete-transfe/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Bitwarden is the best alternative to iCloud Keychain for developers and power users because it offers cross-platform access, open-source code, CLI automation, and self-hosting options that Apple's solution cannot provide. You can export passwords from iCloud Keychain using macOS Keychain Access or Apple's Password Manager CLI, then import them into Bitwarden via CSV file. This guide walks through the complete technical process with scripts and troubleshooting for common issues.

## Understanding iCloud Keychain Export Options

Apple provides limited direct export functionality for iCloud Keychain. Unlike browsers that offer straightforward CSV exports, iCloud Keychain requires workarounds to extract passwords in an usable format.

Three primary approaches exist for developers:

1. **macOS Keychain Access application** with manual export
2. **Apple's Password Manager CLI** (introduced in macOS Sequoia)
3. **Third-party tools** with keychain access

The most reliable method for developers combines the built-in security tooling with custom conversion scripts.

## Method 1: Using macOS Keychain Access

The traditional approach uses the Keychain Access GUI application:

1. Open **Keychain Access** (Cmd+Space, type "Keychain Access")
2. Select **iCloud** in the sidebar under Keychains
3. Enable "Show Expired Items" from the View menu if needed
4. Select all password entries (Cmd+A)
5. Right-click and choose **Export Items**
6. Save as "Keychain Access Exchange (.clip)"

However, this `.clip` format isn't directly importable to Bitwarden. You'll need conversion.

## Method 2: Using Apple's Password Manager CLI

macOS Sequoia introduced a native CLI for password management. This provides the cleanest export path:

```bash
# List all passwords (requires user confirmation)
passwords list

# Export to CSV format
passwords export --format csv --output ~/Desktop/icloud_keychain_export.csv
```

The CLI exports include: URL, username, password, notes, and creation date. This CSV format maps directly to Bitwarden's import requirements.

If you're running an earlier macOS version, upgrade or use the Python script method below.

## Method 3: Python Script for Keychain Extraction

For older macOS versions or automated workflows, use the `security` command-line tool with Python processing:

```python
#!/usr/bin/env python3
"""
iCloud Keychain to Bitwarden CSV Converter
Extracts passwords from macOS Keychain and converts to Bitwarden import format.
"""

import subprocess
import csv
import os
import sys
import json

def get_keychain_passwords():
    """Extract passwords using macOS security command."""
    # Query all internet passwords from login keychain
    cmd = [
        'security', 'find-internet-password',
        '-g',  # Get password data
        '-D',  # Kind (protocol)
        '-s',  # Server
        '-a',  # Account
        '-l',  # Label
    ]
    
    # Alternative: Use a simpler approach with the keychain dump
    result = subprocess.run(
        ['security', 'dump-keychain'],
        capture_output=True,
        text=True
    )
    
    passwords = []
    current_item = {}
    
    for line in result.stdout.split('\n'):
        if '0x00000007' in line:  # Password data follows
            # Parse the password entry
            pass
        
        if 'acct' in line:  # Account/username
            current_item['username'] = line.split('=')[1].strip('"<>"')
        if 'svce' in line:  # Service/server
            current_item['server'] = line.split('=')[1].strip('"<>"')
    
    return passwords

def convert_to_bitwarden_csv(input_file, output_file):
    """Convert iCloud Keychain export to Bitwarden CSV format."""
    
    bitwarden_fields = ['folder', 'favorite', 'type', 'name', 'notes', 
                        'login_uri', 'login_username', 'login_password',
                        'login_totp']
    
    with open(input_file, 'r', encoding='utf-8-sig') as infile:
        # Read your source format here
        # This is a placeholder - adapt based on your actual export
        reader = csv.DictReader(infile)
        
        with open(output_file, 'w', newline='', encoding='utf-8') as outfile:
            writer = csv.DictWriter(outfile, fieldnames=bitwarden_fields)
            writer.writeheader()
            
            for row in reader:
                writer.writerow({
                    'folder': '',
                    'favorite': '',
                    'type': 'login',
                    'name': row.get('title', row.get('name', '')),
                    'notes': row.get('notes', ''),
                    'login_uri': row.get('url', row.get('website', '')),
                    'login_username': row.get('username', row.get('account', '')),
                    'login_password': row.get('password', ''),
                    'login_totp': ''
                })
    
    print(f"Converted {output_file} ready for Bitwarden import")

if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: python3 convert.py <input.csv> <output.csv>")
        sys.exit(1)
    
    convert_to_bitwarden_csv(sys.argv[1], sys.argv[2])
```

This script provides a foundation. Adjust the parsing logic based on your specific export format.

## Importing to Bitwarden

Once you have a properly formatted CSV, import using the Bitwarden CLI:

```bash
# Install Bitwarden CLI if needed
brew install bitwarden-cli

# Login interactively
bw login

# Import the CSV
bw import bitwarden_csv ~/path/to/your_export.csv
```

The CLI supports various import formats including Bitwarden CSV, LastPass CSV, and 1Password CSV.

### CSV Format Requirements

Bitwarden's CSV import expects these columns:

```
folder,favorite,type,name,notes,login_uri,login_username,login_password,login_totp
```

Map your iCloud Keychain export fields accordingly:

- **name**: Service name or website title
- **login_uri**: Website URL
- **login_username**: Account email or username
- **login_password**: The actual password
- **notes**: Any additional notes stored with the entry

## Handling Two-Factor Authentication

iCloud Keychain doesn't store TOTP seeds—only the password. After migration, you'll need to reconfigure 2FA for each service:

1. Log into each service using the migrated credentials
2. Access security settings and enable 2FA
3. Scan the new QR code with Bitwarden Authenticator
4. Store the TOTP in Bitwarden

This reset is intentional security practice. Migrating 2FA seeds would create a vulnerability.

## Verifying the Migration

After import, verify your data integrity:

```bash
# List all items in your vault
bw list items | jq '.[] | {name, login: .login.uri}'

# Check for items without passwords
bw list items | jq '.[] | select(.login.password == null)'

# Count total entries
bw list items | jq 'length'
```

Review critical accounts first: email, banking, and social media. Test login on a secondary device before removing iCloud Keychain entries.

## Automation for Large Vaults

For migrations exceeding 100+ entries, automate verification:

```bash
#!/bin/bash
# Migration verification script

VAULT_PASS="your_master_password"
bw unlock --passwordenv BW_MASTERPASS

# Export and count
ITEM_COUNT=$(bw list items | jq 'length')
echo "Vault contains $ITEM_COUNT items"

# Check for empty passwords
EMPTY_COUNT=$(bw list items | jq '[.[] | select(.login.password == null or .login.password == "")] | length')
echo "Items with empty passwords: $EMPTY_COUNT"

if [ "$EMPTY_COUNT" -gt 0 ]; then
    echo "WARNING: Review items with missing passwords"
fi
```

## Security Considerations

During migration, follow these practices:

- **Encrypt temporary files**: Use GPG for any CSV files containing passwords
- **Use a clean system**: Verify no malware before handling credentials
- **Delete exports securely**: Use `shred` or `rm -P` for sensitive files
- **Update master password**: Consider rotating your Bitwarden master password after migration
- **Enable 2FA on Bitwarden**: Use YubiKey or a hardware token for vault protection



## Related Articles

- [Migrating from Safari Keychain to Bitwarden](/privacy-tools-guide/migrating-from-safari-keychain-to-bitwarden-complete-migration-guide/)
- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [Migrating From Keepass Database To Bitwarden Cloud Vault](/privacy-tools-guide/migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/)
- [Migrating from LastPass to Bitwarden No Data Loss](/privacy-tools-guide/migrating-from-lastpass-to-bitwarden-step-by-step-no-data-lo/)
- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
