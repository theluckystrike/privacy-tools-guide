---
layout: default
title: "Migrating from RoboForm to Bitwarden"
description: "A technical guide for developers and power users migrating password data from RoboForm to Bitwarden, covering export methods, CSV parsing, and import scripts"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /migrating-from-roboform-to-bitwarden-export-import-complete-/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

To migrate from RoboForm to Bitwarden, export your RoboForm vault as CSV from File > Export Data, then import it into Bitwarden using the web vault or CLI. Bitwarden offers open-source code, self-hosting, CLI automation, and full API access that RoboForm cannot match. This guide provides step-by-step instructions with scripts to handle field mapping, duplicate detection, and common migration pitfalls.

## Table of Contents

- [Why Migrate from RoboForm to Bitwarden](#why-migrate-from-roboform-to-bitwarden)
- [Exporting Your RoboForm Data](#exporting-your-roboform-data)
- [Preparing Data for Bitwarden Import](#preparing-data-for-bitwarden-import)
- [Importing into Bitwarden](#importing-into-bitwarden)
- [Handling Complex Data](#handling-complex-data)
- [Post-Migration Checklist](#post-migration-checklist)
- [Security Considerations During Migration](#security-considerations-during-migration)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Why Migrate from RoboForm to Bitwarden

RoboForm served many users well for years, but several factors drive developers and power users toward Bitwarden:

- **Open-source architecture**: Bitwarden's client code is publicly auditable
- **Self-hosting option**: Run your own vault server for complete data control
- **CLI excellence**: The Bitwarden CLI is and scriptable
- **API access**: Full programmatic access to vault operations
- **Import flexibility**: Supports more formats than most competitors

If you've outgrown RoboForm's limited CLI and lack of API access, the migration path is straightforward with the right approach.

## Exporting Your RoboForm Data

RoboForm provides export functionality through both the desktop application and web interface. The process generates a CSV file containing your vault data.

### Using the RoboForm Desktop Application

1. Open RoboForm Desktop on Windows or macOS
2. Navigate to File → Export Data
3. Select "All Items" or choose specific folders
4. Choose CSV format (avoid the encrypted RoboForm format for migration)
5. Save the file to a secure location

### Using RoboForm Online

1. Log into your RoboForm account at web.roboform.com
2. Click your profile icon → Settings
3. Scroll to Data → Export
4. Download your data as CSV

The exported CSV contains columns for: name, login, password, url, note, and folder. Verify the file opens correctly in a spreadsheet application before proceeding.

## Preparing Data for Bitwarden Import

Bitwarden's import functionality expects specific CSV formats. The RoboForm export requires transformation to match Bitwarden's expected structure.

### Bitwarden Import Format

Bitwarden accepts CSV with these required columns:
- `name` (required): Entry name
- `login_uri` (required): Website URL
- `login_username`: Account username or email
- `login_password`: The password
- `notes`: Additional notes

Optional columns include `favorite`, `login_totp`, and custom field support.

### Python Script for Conversion

Here's a practical conversion script:

```python
#!/usr/bin/env python3
import csv
import sys
import argparse

def convert_roboform_to_bitwarden(input_file, output_file):
    """
    Convert RoboForm CSV export to Bitwarden CSV format.
    """
    with open(input_file, 'r', encoding='utf-8-sig') as f_in:
        reader = csv.DictReader(f_in)

        # Write Bitwarden format
        with open(output_file, 'w', newline='', encoding='utf-8') as f_out:
            fieldnames = ['name', 'login_uri', 'login_username', 'login_password', 'notes']
            writer = csv.DictWriter(f_out, fieldnames=fieldnames)
            writer.writeheader()

            for row in reader:
                # Skip empty passwords
                if not row.get('password'):
                    continue

                # Build Bitwarden entry
                entry = {
                    'name': row.get('name', 'Unnamed'),
                    'login_uri': row.get('url', ''),
                    'login_username': row.get('login', ''),
                    'login_password': row.get('password', ''),
                    'notes': row.get('note', '')
                }
                writer.writerow(entry)

    print(f"Converted {input_file} to {output_file}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Convert RoboForm export to Bitwarden format')
    parser.add_argument('input', help='Input CSV file')
    parser.add_argument('output', help='Output CSV file')
    args = parser.parse_args()

    convert_roboform_to_bitwarden(args.input, args.output)
```

Run the conversion:

```bash
python3 convert_roboform.py roboform_export.csv bitwarden_import.csv
```

### Handling Special Characters

RoboForm sometimes exports passwords containing commas or quotes. The script handles CSV escaping, but verify problematic entries after conversion. Check for entries where passwords contain newline characters or unusual encoding.

## Importing into Bitwarden

### Via Web Interface

1. Log into vault.bitwarden.com
2. Navigate to Tools → Import Data
3. Select "Bitwarden (csv)" as format
4. Upload your converted CSV
5. Click Import

### Via Bitwarden CLI

The CLI provides more control and works well for automated pipelines:

```bash
# Install Bitwarden CLI
brew install bitwarden-cli

# Login interactively
bw login your-email@example.com

# Unlock vault (will prompt for master password)
bw unlock

# Import the CSV
bw import bitwarden bitwarden_import.csv
```

### Verifying Import Success

After import, verify your data:

```bash
# List all items
bw list items | jq '.[] | .name'

# Check item count matches expected
bw list items | jq 'length'
```

## Handling Complex Data

### Secure Notes

RoboForm's secure notes convert to Bitwarden's notes field. If you stored structured data in notes (like API keys or server configurations), consider parsing them into Bitwarden's custom fields using an extended script:

```python
def parse_custom_fields(notes_text):
    """
    Parse structured data from notes into Bitwarden custom fields.
    Returns list of field dictionaries.
    """
    fields = []
    lines = notes_text.split('\n')

    for line in lines:
        if ':' in line:
            key, value = line.split(':', 1)
            fields.append({
                'name': key.strip(),
                'value': value.strip(),
                'type': 0  # 0 = text, 1 = hidden
            })

    return fields
```

### Folders and Collections

RoboForm's folder structure maps to Bitwarden collections in paid plans. Free tier users lose folder hierarchy—entries import without folder assignment. For folder preservation:

1. Use Bitwarden Premium or Families
2. Create collections matching RoboForm folders
3. Post-process with CLI to assign entries

```bash
# Create collection
bw create collection '{"name": "Work Accounts", "organizationId": "YOUR_ORG_ID"}'

# Move item to collection (requires organization setup)
bw move <item_id> <collection_id>
```

### Identities and Form Data

RoboForm's identity profiles (personal info, addresses, credit cards) require separate handling. Bitwarden handles identities differently—create separate identity items rather than combining with login entries.

## Post-Migration Checklist

After migrating, verify your new setup:

1. **Test login entries**: Confirm a sample of passwords work at their respective sites
2. **Verify TOTP codes**: If using RoboForm's authenticator, export and import to Bitwarden or your preferred TOTP app
3. **Check browser extensions**: Re-authenticate Bitwarden extensions on all browsers
4. **Update CLI configurations**: Reconfigure any scripts using Bitwarden CLI
5. **Export Bitwarden backup**: Create a fresh backup of your new vault
6. **Secure the old export**: Delete or encrypt the RoboForm CSV containing plaintext passwords

## Security Considerations During Migration

The migration process involves handling sensitive data in plaintext. Follow these practices:

- **Use an air-gapped machine** for conversion if possible
- **Encrypt the CSV** after import confirmation: `gpg -c bitwarden_import.csv`
- **Delete temporary files** immediately after migration
- **Use CLI over web interface** for reduced exposure to browser extensions

## Troubleshooting Common Issues

### "Invalid CSV format" Errors

Bitwarden is strict about CSV formatting. Common issues:
- Missing required columns (name, login_uri, login_username)
- UTF-8 encoding problems—always use `encoding='utf-8-sig'` when reading
- Empty rows in CSV causing parsing failures

### Duplicate Entries

If importing multiple times, Bitwarden creates duplicates. Use the CLI's `--force` flag carefully or manually deduplicate:

```bash
# List duplicates by name
bw list items | jq 'group_by(.name) | map(select(length > 1))'
```

### Missing Passwords

Some RoboForm entries might have empty password fields (like WiFi passwords or secure notes). These won't appear in Bitwarden's login items—manually recreate them or import as secure notes.

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

- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)
- [Migrating from LastPass to Bitwarden No Data Loss](/privacy-tools-guide/migrating-from-lastpass-to-bitwarden-step-by-step-no-data-lo/)
- [Migrating From Keepass Database To Bitwarden Cloud Vault](/privacy-tools-guide/migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/)
- [Bitwarden vs LastPass Migration Guide](/privacy-tools-guide/bitwarden-vs-lastpass-migration-guide/)
- [Migrating From Icloud Keychain To Bitwarden Complete](/privacy-tools-guide/migrating-from-icloud-keychain-to-bitwarden-complete-transfe/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
