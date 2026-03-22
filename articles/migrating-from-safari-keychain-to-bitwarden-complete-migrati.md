---
---
layout: default
title: "Migrating from Safari Keychain to Bitwarden"
description: "A technical guide for developers and power users migrating passwords from Safari Keychain to Bitwarden. Covers CLI export, CSV conversion, and automation"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /migrating-from-safari-keychain-to-bitwarden-complete-migration-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Apple's Safari Keychain provides basic password management for macOS and iOS users, but developers and power users often need more advanced features. Bitwarden offers open-source transparency, cross-platform support, CLI tooling, and self-hosting options. This guide covers the technical process of moving your credentials from Safari Keychain to Bitwarden using command-line methods.

## Key Takeaways

- **Free tiers typically have**: usage limits that work for evaluation but may not be sufficient for daily professional use.
- **Does Bitwarden offer a**: free tier? Most major tools offer some form of free tier or trial period.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **Bitwarden offers open-source transparency,**: cross-platform support, CLI tooling, and self-hosting options.
- **Mastering advanced features takes**: 1-2 weeks of regular use.
- **Focus on the 20%**: of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Understanding Safari Keychain Export Limitations

Safari stores passwords in the macOS Keychain, which uses a proprietary format. Apple does not provide a direct export feature through its graphical interface, forcing users to rely on workarounds.

The primary challenge is accessing the Keychain in a format Bitwarden can import. Safari Keychain entries contain the website URL, username, password, and notes field. However, the export process requires either manual extraction or third-party tools.

For developers comfortable with command-line tools, several approaches exist:

1. **Using the security command-line tool** (macOS built-in)
2. **Python scripts** for parsing Keychain exports
3. **CSV conversion workflows** for Bitwarden import

## Method 1: Using the macOS Security Command

The `security` command provides access to Keychain data programmatically. Create a script to extract all Safari passwords:

```bash
#!/bin/bash
# extract-safari-keychain.sh

OUTPUT_FILE="safari_passwords.csv"
echo "url,username,password,notes" > "$OUTPUT_FILE"

# Get all Safari passwords
security find-internet-password -s safari -g 2>/dev/null | \
while read -r line; do
    if [[ "$line" == *"server:"* ]]; then
        url=$(echo "$line" | sed 's/.*server: \(.*\)/\1/')
    elif [[ "$line" == *"acct:"* ]]; then
        username=$(echo "$line" | sed 's/.*acct: <.*> \(.*\)/\1/')
    elif [[ "$line" == *"password:"* ]]; then
        password=$(echo "$line" | sed 's/.*password: "\(.*\)"/\1/')
        echo "$url,$username,$password," >> "$OUTPUT_FILE"
    fi
done
```

This script queries the login Keychain for Safari credentials and formats them as CSV.

## Method 2: Python-Based Extraction with Keyring

For more reliable extraction, use Python with the `keyring` library:

```python
#!/usr/bin/env python3
# extract_keychain.py

import keyring
import csv
import subprocess

def get_safari_passwords():
    """Extract Safari passwords from Keychain"""
    passwords = []

    # Query Keychain for Safari entries
    result = subprocess.run(
        ['security', 'dump-keychain'],
        capture_output=True,
        text=True
    )

    # Parse the output for Safari passwords
    current_entry = {}
    for line in result.stdout.split('\n'):
        if 'server' in line.lower():
            current_entry['url'] = line.split('=')[-1].strip().strip('"')
        elif 'acct' in line.lower():
            current_entry['username'] = line.split('=')[-1].strip().strip('"')
        elif 'password' in line.lower() and 'password\00' not in line.lower():
            # Extract password value
            current_entry['password'] = line.split('=')[-1].strip().strip('"')

            if current_entry.get('url') and 'safari' in current_entry.get('url', '').lower():
                passwords.append(current_entry)
            current_entry = {}

    return passwords

def export_to_csv(passwords, filename='bitwarden_import.csv'):
    """Export to Bitwarden-compatible CSV format"""
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['folder', 'favorite', 'type', 'name', 'notes',
                        'fields', 'login_uri', 'login_username',
                        'login_password', 'login_totp'])

        for entry in passwords:
            writer.writerow([
                '',  # folder
                '',  # favorite
                'login',  # type
                entry.get('url', ''),  # name
                '',  # notes
                '',  # custom fields
                entry.get('url', ''),  # URI
                entry.get('username', ''),  # username
                entry.get('password', ''),  # password
                ''  # TOTP
            ])

if __name__ == '__main__':
    passwords = get_safari_passwords()
    export_to_csv(passwords)
    print(f"Exported {len(passwords)} passwords to bitwarden_import.csv")
```

Install dependencies with:

```bash
pip install keyring
```

## Method 3: Using Brett Terpstra's Export Script

Many macOS users rely on Brett Terpstra's [Safari Password Exporter](https://github.com/tmestd/Safari-Password-Exporter) script, which provides a GUI for Keychain extraction. For command-line enthusiasts, the underlying mechanism works similarly.

Clone and run the exporter:

```bash
git clone https://github.com/tmestd/Safari-Password-Exporter.git
cd Safari-Password-Exporter
chmod +x SafariPasswordExporter.py
./SafariPasswordExporter.py
```

The tool generates a CSV file compatible with most password managers.

## Preparing Data for Bitwarden Import

Bitwarden accepts CSV imports with specific column headers. Ensure your export follows this format:

| Column | Description |
|--------|-------------|
| folder | Optional folder name |
| favorite | 0 or 1 |
| type | "login" for passwords |
| name | Entry title |
| notes | Additional notes |
| login_uri | Website URL |
| login_username | Username or email |
| login_password | Password |
| login_totp | TOTP seed (optional) |

Clean your CSV before import:

```bash
# Remove empty lines and validate format
awk -F',' 'NF >= 8' safari_export.csv > bitwarden_import.csv
```

## Importing into Bitwarden

### Using the Web Vault

1. Log into vault.bitwarden.com
2. Navigate to **Settings** → **Import Data**
3. Select **Bitwarden (csv)** as the format
4. Upload your CSV file
5. Click **Import**

### Using Bitwarden CLI

For automation, use the Bitwarden CLI:

```bash
# Install CLI
brew install bitwarden-cli

# Login
bw login your@email.com

# Unlock vault
bw unlock

# Import the CSV
bw import bitwarden_csv bitwarden_import.csv
```

The CLI requires authentication, so you'll need to complete the login flow interactively or use environment variables for scripted environments.

## Handling TOTP Codes

Safari Keychain does not store TOTP seeds, so you'll need to regenerate 2FA codes for each service. This is actually a security feature—migrating 2FA to a new device requires re-setup for each account.

For services requiring 2FA:

1. Generate new TOTP codes in Bitwarden
2. Update each account's 2FA settings with the new code
3. Test login on a secondary device before removing old 2FA

Bitwarden's TOTP integration works with Google Authenticator, Authy, and hardware tokens.

## Verification and Cleanup

After import, verify your data:

```bash
# List imported items using CLI
bw list items --folderid [folder-id] | jq '.[] | {name, login}'
```

Check for:
- Duplicate entries
- Missing passwords
- Incorrect URLs
- Empty fields

Delete the temporary CSV files afterward:

```bash
rm -f safari_export.csv bitwarden_import.csv
```

## Security Considerations

When migrating passwords, follow these security practices:

- **Use a clean machine**: Ensure no malware exists before handling credentials
- **Encrypt exports**: If storing the CSV temporarily, encrypt it with GPG
- **Verify checksums**: Confirm file integrity before deletion
- **Update master password**: Consider rotating your Bitwarden master password after migration

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

- [Migrating From Icloud Keychain To Bitwarden Complete Transfe](/privacy-tools-guide/migrating-from-icloud-keychain-to-bitwarden-complete-transfe/)
- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [Migrating From Keepass Database To Bitwarden Cloud Vault](/privacy-tools-guide/migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/)
- [Migrating from LastPass to Bitwarden No Data Loss](/privacy-tools-guide/migrating-from-lastpass-to-bitwarden-step-by-step-no-data-lo/)
- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
