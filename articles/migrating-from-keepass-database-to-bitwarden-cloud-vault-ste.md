---
layout: default
title: "Migrating From Keepass Database To Bitwarden Cloud Vault"
description: "A technical guide for developers and power users moving from local KeePass databases to Bitwarden's cloud vault. Covers export methods, CLI tools, and"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---
---
layout: default
title: "Migrating From Keepass Database To Bitwarden Cloud Vault"
description: "A technical guide for developers and power users moving from local KeePass databases to Bitwarden's cloud vault. Covers export methods, CLI tools, and"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /migrating-from-keepass-database-to-bitwarden-cloud-vault-step-by-step/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true---

{% raw %}

Migrating your passwords from a local KeePass database to Bitwarden's cloud vault is a practical upgrade if you need cross-device synchronization, easier sharing with family or team members, or a more accessible backup solution. This guide walks you through the entire process using command-line tools and automation scripts, ideal for developers who prefer terminal-based workflows.

## Key Takeaways

- **Free tiers typically have**: usage limits that work for evaluation but may not be sufficient for daily professional use.
- **If you used a**: separate TOTP authenticator, you'll need to re-add 2FA to your Bitwarden entries manually or export from your TOTP app if supported.
- **Does Bitwarden offer a**: free tier? Most major tools offer some form of free tier or trial period.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **Mastering advanced features takes**: 1-2 weeks of regular use.
- **Focus on the 20%**: of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Why Migrate From KeePass to Bitwarden

KeePass and KeePassXC store your password database in an encrypted file (`.kdbx`) on your local machine. This approach gives you full control over your data but lacks native cloud synchronization. You likely sync your database manually or through a service like Dropbox, which can create version conflicts and leaves your vault vulnerable if your sync folder is compromised.

Bitwarden stores your encrypted vault on their servers while maintaining zero-knowledge encryption—your master password never leaves your device. The cloud approach means your passwords are available instantly on any device without manual file management.

## Prerequisites

Before starting the migration, ensure you have:

- A KeePass or KeePassXC database with its master key
- A Bitwarden account (free tier works for personal vaults)
- Python 3.8+ installed (for the migration script)
- The Bitwarden CLI installed

Install the Bitwarden CLI:

```bash
# macOS
brew install bitwarden-cli

# Ubuntu/Debian
sudo apt install bitwarden

# Or download from https://github.com/bitwarden/cli/releases
```

## Step 1: Export Your KeePass Database

KeePass provides a built-in export feature, but for programmatic access, you have two reliable options.

### Option A: Using KeePassXC CLI

KeePassXC includes `keepassxc-cli` for command-line operations:

```bash
# Export database to XML (unencrypted)
keepassxc-cli export -o keepass_export.xml your_database.kdbx
```

You'll be prompted for your master password. The XML output contains all entries with fields like username, password, URL, notes, and custom attributes.

### Option B: Using Python with pykeepass

For more control, use the `pykeepass` library:

```bash
pip install pykeepass
```

Create a Python script to extract your entries:

```python
from pykeepass import PyKeePass

kp = PyKeePass('your_database.kdbx', password='your_master_password')

entries = []
for entry in kp.entries:
    entries.append({
        'title': entry.title,
        'username': entry.username,
        'password': entry.password,
        'url': entry.url,
        'notes': entry.notes,
        'tags': entry.tags
    })

# Output to JSON for easier parsing
import json
with open('keepass_export.json', 'w') as f:
    json.dump(entries, f, indent=2)
```

This approach preserves custom fields and allows selective migration based on tags or folders.

## Step 2: Authenticate with Bitwarden CLI

Initialize the Bitwarden CLI and log in:

```bash
bw login your@email.com
```

You'll receive a code to complete authentication via browser. For CI/CD environments, use an API key:

```bash
bw config apiurl https://api.bitwarden.com
bw login --apikey
```

Set up your encryption key:

```bash
bw unlock
```

The CLI returns a session key—store this securely. You can also use the `BW_SESSION` environment variable:

```bash
export BW_SESSION="your_session_key"
```

## Step 3: Import Entries to Bitwarden

Bitwarden supports CSV import, which works well for most migrations. Generate a CSV from your export:

```python
import json
import csv

with open('keepass_export.json', 'r') as f:
    entries = json.load(f)

with open('bitwarden_import.csv', 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['title', 'username', 'password', 'url', 'notes', 'folder'])

    for entry in entries:
        folder = entry.get('tags', [''])[0] if entry.get('tags') else ''
        writer.writerow([
            entry.get('title', ''),
            entry.get('username', ''),
            entry.get('password', ''),
            entry.get('url', ''),
            entry.get('notes', ''),
            folder
        ])
```

Import using the Bitwarden CLI:

```bash
bw import bitwarden_import.csv --formats keepass
```

The `--formats keepass` flag tells Bitwarden to map KeePass fields correctly.

## Step 4: Automate the Full Migration

For a complete automation solution, here's a consolidated Python script:

```python
#!/usr/bin/env python3
import os
import json
import csv
import subprocess
import sys
from pykeepass import PyKeePass

def export_keepass(db_path, password, output_json):
    """Export KeePass database to JSON."""
    kp = PyKeePass(db_path, password=password)
    entries = []

    for entry in kp.entries:
        entries.append({
            'title': entry.title,
            'username': entry.username,
            'password': entry.password,
            'url': entry.url,
            'notes': entry.notes,
            'tags': entry.tags,
            'custom_fields': [
                {'name': cf.key, 'value': cf.value}
                for cf in entry.custom_properties
            ]
        })

    with open(output_json, 'w') as f:
        json.dump(entries, f, indent=2)

    return len(entries)

def json_to_csv(json_file, csv_file):
    """Convert JSON export to Bitwarden CSV format."""
    with open(json_file, 'r') as f:
        entries = json.load(f)

    with open(csv_file, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['title', 'username', 'password', 'url', 'notes', 'folder'])

        for entry in entries:
            folder = entry.get('tags', [''])[0] if entry.get('tags') else ''
            writer.writerow([
                entry.get('title', ''),
                entry.get('username', ''),
                entry.get('password', ''),
                entry.get('url', ''),
                entry.get('notes', ''),
                folder
            ])

def import_to_bitwarden(csv_file):
    """Import CSV to Bitwarden using CLI."""
    result = subprocess.run(
        ['bw', 'import', csv_file, '--formats', 'keepass'],
        capture_output=True, text=True
    )
    return result.returncode == 0

def main():
    if len(sys.argv) != 4:
        print("Usage: migrate.py <keepass.db> <password> <output_dir>")
        sys.exit(1)

    db_path, password, output_dir = sys.argv[1], sys.argv[2], sys.argv[3]
    os.makedirs(output_dir, exist_ok=True)

    json_file = os.path.join(output_dir, 'keepass_export.json')
    csv_file = os.path.join(output_dir, 'bitwarden_import.csv')

    print(f"Exporting {db_path}...")
    count = export_keepass(db_path, password, json_file)
    print(f"Exported {count} entries")

    print("Converting to CSV...")
    json_to_csv(json_file, csv_file)

    print("Importing to Bitwarden...")
    if import_to_bitwarden(csv_file):
        print("Migration completed successfully!")
    else:
        print("Import failed. Check Bitwarden CLI authentication.")

if __name__ == '__main__':
    main()
```

Run the migration:

```bash
python migrate.py your_database.kdbx "your_master_password" ./migration_output
```

## Step 5: Verify and Clean Up

After migration, verify your entries in the Bitwarden web vault or desktop app:

1. Check that passwords match between old and new entries
2. Verify custom fields transferred correctly
3. Ensure folders/tags mapped properly
4. Test login with a few critical accounts

Delete the temporary export files securely:

```bash
# macOS
srm keepass_export.json bitwarden_import.csv

# Linux
shred -u keepass_export.json bitwarden_import.csv
```

## Handling Edge Cases

**Two-Factor Authentication**: KeePass doesn't store TOTP seeds in the standard entry fields. If you used a separate TOTP authenticator, you'll need to re-add 2FA to your Bitwarden entries manually or export from your TOTP app if supported.

**Custom Attributes**: KeePass allows arbitrary key-value pairs on entries. Bitwarden's custom fields support this, but the Python script handles basic custom properties. Review entries with complex custom fields after import.

**Database Merging**: If you have multiple KeePass databases, run the migration script for each and import the resulting CSVs sequentially. Bitwarden handles duplicates based on title and username.

## Security Considerations

During migration, your passwords briefly exist in unencrypted files. Work in an isolated environment:

- Use a fresh temporary directory
- Close other password managers
- Clear clipboard after use
- Delete temporary files immediately after verification

Bitwarden's zero-knowledge architecture means even their servers cannot read your vault. Your master password is the only key—ensure it's strong and unique.

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
- [Migrating from LastPass to Bitwarden No Data Loss](/privacy-tools-guide/migrating-from-lastpass-to-bitwarden-step-by-step-no-data-lo/)
- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)
- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [Migrating from Safari Keychain to Bitwarden](/privacy-tools-guide/migrating-from-safari-keychain-to-bitwarden-complete-migration-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
