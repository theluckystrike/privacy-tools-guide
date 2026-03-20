---
layout: default
title: "Migrating from LastPass to Bitwarden No Data Loss"
description: "A technical guide for developers and power users migrating from LastPass to Bitwarden without losing any passwords, TOTP codes, or custom."
date: 2026-03-16
author: theluckystrike
permalink: /migrating-from-lastpass-to-bitwarden-step-by-step-no-data-lo/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Migrating from LastPass to Bitwarden Step by Step: No Data Loss

Switching password managers doesn't have to mean losing your data. With the right approach, you can transfer every login, secure note, and TOTP code from LastPass to Bitwarden without gaps. This guide covers the technical steps for developers and power users who want full control over their migration.

## Prerequisites

Before starting the migration, ensure you have:

- A LastPass account with your master password
- A Bitwarden account (free tier works for personal vaults)
- LastPass browser extension or CLI installed
- Bitwarden CLI installed (`bw` command)

Install the Bitwarden CLI via your package manager:

```bash
# macOS
brew install bitwarden-cli

# Ubuntu/Debian
sudo apt install bitwarden

# npm
npm install -g @bitwarden/cli
```

## Step 1: Export Data from LastPass

LastPass provides two export formats: CSV and encrypted JSON. For a complete migration, use the CSV export as it preserves more fields.

### Using the LastPass CLI

If you have `lpass` installed, export your vault:

```bash
lpass export --format=csv > lastpass-export.csv
```

### Using the Web Interface

1. Log into LastPass at lastpass.com
2. Go to **Advanced Options** > **Export**
3. Choose **LastPass CSV File**
4. Save the file to your local machine

Review the exported file to ensure all entries are present:

```bash
wc -l lastpass-export.csv
head -5 lastpass-export.csv
```

The CSV contains columns for URL, username, password, name, and notes.

## Step 2: Handle TOTP Codes

LastPass stores TOTP secrets within the notes field or as separate entries. Extract these before migration:

```bash
# Extract entries with TOTP in notes
grep -i "otp" lastpass-export.csv | head -10
```

If you have many TOTP codes, use a Python script to extract and convert them:

```python
import csv
import re

def extract_totp(notes):
    """Extract TOTP secret from LastPass notes field."""
    match = re.search(r'otpauth://[^"\s]+', notes)
    if match:
        url = match.group(0)
        secret = re.search(r'secret=([A-Z0-9]+)', url)
        if secret:
            return secret.group(1)
    return None

# Process CSV and identify TOTP entries
with open('lastpass-export.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        totp = extract_totp(row.get('notes', ''))
        if totp:
            print(f"{row['name']}: {totp}")
```

## Step 3: Import into Bitwarden

Bitwarden supports direct CSV import with its CLI. First, unlock your vault:

```bash
bw unlock
```

This returns a session key. Set it as an environment variable:

```bash
export BW_SESSION="your-session-key-here"
```

### Format the Import File

Bitwarden expects a specific CSV format. Create a conversion script:

```python
import csv

def convert_to_bitwarden(input_file, output_file):
    """Convert LastPass export to Bitwarden format."""
    field_map = {
        'url': 'login_uri',
        'username': 'login_username', 
        'password': 'login_password',
        'name': 'name',
        'notes': 'notes'
    }
    
    with open(input_file, 'r', encoding='utf-8') as f_in:
        reader = csv.DictReader(f_in)
        
        with open(output_file, 'w', newline='', encoding='utf-8') as f_out:
            writer = csv.DictWriter(f_out, fieldnames=field_map.keys())
            writer.writeheader()
            
            for row in reader:
                # Map LastPass fields to Bitwarden format
                converted = {k: row.get(v, '') for k, v in field_map.items()}
                # Ensure URL has proper prefix
                if converted['login_uri'] and not converted['login_uri'].startswith(('http://', 'https://')):
                    converted['login_uri'] = 'https://' + converted['login_uri']
                writer.writerow(converted)

convert_to_bitwarden('lastpass-export.csv', 'bitwarden-import.csv')
```

### Run the Import

```bash
bw import --format=csv --file=bitwarden-import.csv
```

Verify the import count matches your export:

```bash
bw list items | jq length
```

## Step 4: Import TOTP Codes

Bitwarden stores TOTP as a separate field. Add TOTP codes programmatically:

```python
import subprocess
import csv
import re
import json

def add_totp_to_item(item_id, totp_secret):
    """Add TOTP code to existing Bitwarden item."""
    # First, get the current item
    result = subprocess.run(
        ['bw', 'get', 'item', item_id],
        capture_output=True, text=True
    )
    item = json.loads(result.stdout)
    
    # Add TOTP to the item
    item['login']['totp'] = totp_secret
    
    # Save back
    subprocess.run(
        ['bw', 'edit', 'item', item_id],
        input=json.dumps(item), text=True
    )

# Process TOTP entries
with open('lastpass-export.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        # Extract and add TOTP codes
        pass  # Implementation based on your extraction logic
```

Alternatively, use Bitwarden's web vault to manually add TOTP codes for critical accounts like email and banking.

## Step 5: Verify Migration Completeness

Run comparison checks to ensure nothing was lost:

```bash
# Count LastPass entries
echo "LastPass entries: $(tail -n +2 lastpass-export.csv | wc -l)"

# Count Bitwarden entries  
echo "Bitwarden entries: $(bw list items | jq length)"

# List Bitwarden folders
bw list folders

# List Bitwarden collections
bw list collections
```

Create a verification script:

```bash
#!/bin/bash

# Compare entry counts
LP_COUNT=$(tail -n +2 lastpass-export.csv | wc -l)
BW_COUNT=$(bw list items | jq '. | length')

if [ "$LP_COUNT" -eq "$BW_COUNT" ]; then
    echo "✓ Entry count matches: $LP_COUNT"
else
    echo "✗ Mismatch: LastPass has $LP_COUNT, Bitwarden has $BW_COUNT"
fi
```

## Step 6: Configure Bitwarden CLI for Automation

Set up CLI shortcuts for daily use:

```bash
# Quick unlock with environment variable
export BW_CLIENTID="your-client-id"
export BW_CLIENTSECRET="your-client-secret"

# Sync vault
bw sync

# Generate password
bw generate --length 24 --uppercase --lowercase --number --symbol
```

Add to your shell profile for persistent access:

```bash
echo 'export BW_SESSION="$(bw unlock --raw)"' >> ~/.bashrc
```

## Step 7: Browser Extension and Migration

Install the Bitwarden browser extension:

- Chrome/Chromium: [Chrome Web Store](https://chrome.google.com/webstore/detail/bitwarden-free-password-m/nngceckbapebfimnlniiiahkandclblb)
- Firefox: [Add-ons for Firefox](https://addons.mozilla.org/en-US/firefox/addon/bitwarden-password-manager/)
- Safari: Mac App Store

After installation:

1. Sign in with your Bitwarden credentials
2. Enable auto-fill in extension settings
3. Check "Disable auto-fill on page load" to prevent conflicts
4. Import browser passwords from your browser's built-in manager

## Troubleshooting Common Issues

### Encoding Problems

If special characters don't import correctly:

```python
# Ensure UTF-8 encoding
with open('file.csv', 'r', encoding='utf-8-sig') as f:
    # Process with proper encoding
```

### Duplicate Entries

Bitwarden prevents exact duplicates. Check for them before import:

```bash
sort bitwarden-import.csv | uniq -d
```

### Missing Custom Fields

LastPass custom fields need manual recreation:

```bash
# List items with notes containing custom field markers
grep -i "custom field" lastpass-export.csv
```

Re-add these manually through the Bitwarden web interface or CLI.

## Security Considerations

After migration, strengthen your security:

1. **Rotate critical passwords**: Prioritize email, banking, and social media
2. **Enable two-factor authentication**: Use YubiKey or TOTP
3. **Audit login history**: Check for unauthorized access
4. **Delete LastPass data**: Remove your vault after confirming migration success
5. **Update recovery options**: Ensure your email and phone are current

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
