---
layout: default
title: "How to Audit Your Password Manager Vault: A Practical Guide"
description: "Learn how to audit your password manager vault with CLI tools and automated scripts. Find weak passwords, reused credentials, and security gaps"
date: 2026-03-15
author: theluckystrike
permalink: /how-to-audit-your-password-manager-vault/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Regularly auditing your password manager vault is essential for maintaining strong security hygiene. Over time, vaults accumulate stale credentials, weak passwords, and forgotten shared items that create attack vectors. This guide walks through practical methods to audit your vault using command-line tools and scripts, designed for developers and power users who want automation and control.

## Why Vault Auditing Matters

Password managers simplify credential management, but they can also become repositories of security debt. A vault with hundreds of entries likely contains:

- Weak passwords that haven't been updated in years
- Reused passwords across multiple services
- Accounts for services that no longer exist
- Stale two-factor authentication methods
- Outdated recovery options

An audit helps you identify and remediate these issues before they become problems. Rather than manually reviewing each entry, you can use CLI tools and scripting to automate much of this process.

## Exporting Your Vault Data

The first step in any audit is getting your vault data into a format you can analyze. Most password managers support encrypted exports that maintain your data's security during the audit process.

### Bitwarden CLI Export

Bitwarden provides a straightforward export command:

```bash
# Install Bitwarden CLI first
npm install -g @bitwarden/cli

# Login and unlock
bw login your-email@example.com
bw unlock

# Export to JSON (will prompt for encryption password)
bw export --output ~/vault-export.json --format json
```

The exported JSON contains all your vault items including login credentials, secure notes, and payment cards. Keep this file secure—it's unencrypted unless you provide a password during export.

### 1Password CLI Export

1Password requires using the CLI to export data:

```bash
# Install 1Password CLI
brew install --cask 1password-cli

# Sign in
op signin

# Export vault to JSON
op list items | op get item - > ~/1password-export.json
```

### KeePassXC Export

For KeePass databases, you can export directly:

```bash
# Using keepassxc-cli
keepassxc-cli export --format csv database.kdbx -o vault-export.csv
```

## Analyzing Password Strength

Once you have your vault data exported, the next step is analyzing password strength. You can write custom scripts or use existing tools to identify weak passwords.

### Python Script for Password Analysis

Here's a practical script that analyzes exported Bitwarden JSON:

```python
#!/usr/bin/env python3
import json
import re
import sys
from typing import Dict, List

def analyze_password(password: str) -> Dict[str, any]:
    """Analyze a password for common weaknesses."""
    result = {
        'length': len(password),
        'has_upper': bool(re.search(r'[A-Z]', password)),
        'has_lower': bool(re.search(r'[a-z]', password)),
        'has_digit': bool(re.search(r'[0-9]', password)),
        'has_special': bool(re.search(r'[!@#$%^&*(),.?":{}|<>]', password)),
        'is_weak': False,
        'issues': []
    }
    
    if len(password) < 12:
        result['is_weak'] = True
        result['issues'].append('password too short (<12 characters)')
    
    if not result['has_upper'] or not result['has_lower']:
        result['is_weak'] = True
        result['issues'].append('missing case variation')
    
    if not result['has_digit'] or not result['has_special']:
        result['is_weak'] = True
        result['issues'].append('missing character diversity')
    
    # Check for common patterns
    if re.match(r'^[a-zA-Z]+$', password):
        result['issues'].append('letters only')
    if re.match(r'^[0-9]+$', password):
        result['issues'].append('digits only')
    
    return result

def audit_vault(json_file: str):
    """Audit exported Bitwarden vault."""
    with open(json_file, 'r') as f:
        vault = json.load(f)
    
    weak_passwords = []
    items = vault.get('items', [])
    
    for item in items:
        if item.get('type') != 1:  # Skip non-login items
            continue
            
        login = item.get('login', {})
        password = login.get('password', '')
        
        if not password:
            continue
            
        analysis = analyze_password(password)
        
        if analysis['is_weak']:
            weak_passwords.append({
                'name': item.get('name', 'Unknown'),
                'url': login.get('uri', ''),
                'issues': analysis['issues']
            })
    
    # Report results
    print(f"\nVault Audit Results")
    print(f"=" * 50)
    print(f"Total login items: {len(items)}")
    print(f"Weak passwords found: {len(weak_passwords)}\n")
    
    if weak_passwords:
        print("Weak Passwords:")
        for item in weak_passwords:
            print(f"  - {item['name']} ({item['url']})")
            for issue in item['issues']:
                print(f"      {issue}")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python audit_vault.py <exported-json-file>")
        sys.exit(1)
    audit_vault(sys.argv[1])
```

Run this script against your exported vault:

```bash
python3 audit_vault.py ~/vault-export.json
```

## Identifying Reused Passwords

Password reuse is one of the most common security vulnerabilities. A data breach at one service exposes all accounts using the same password.

### Finding Duplicates

Add this function to your analysis script to detect password reuse:

```python
def find_reused_passwords(vault_data: Dict) -> List[Dict]:
    """Find passwords used across multiple accounts."""
    password_map = {}
    
    for item in vault_data.get('items', []):
        if item.get('type') != 1:
            continue
        
        password = item.get('login', {}).get('password', '')
        if not password:
            continue
            
        if password not in password_map:
            password_map[password] = []
        
        password_map[password].append({
            'name': item.get('name'),
            'url': item.get('login', {}).get('uri', '')
        })
    
    # Return only duplicates
    return [
        {'password': p, 'accounts': accounts}
        for p, accounts in password_map.items()
        if len(accounts) > 1
    ]
```

This identifies which passwords appear multiple times in your vault, allowing you to prioritize updating these credentials.

## Checking for Compromised Passwords

Beyond analyzing password strength, you should check whether your passwords have appeared in known data breaches. The Have I Been Pwned API provides this functionality through a k-anonymity model that protects your actual passwords.

### Integration with HIBP

```python
import hashlib
import requests

def check_compromised(password: str) -> bool:
    """Check if password appears in known breaches."""
    # Hash password with SHA-1
    sha1_hash = hashlib.sha1(password.encode('utf-8')).hexdigest().upper()
    prefix, suffix = sha1_hash[:5], sha1_hash[5:]
    
    # Query HIBP API with prefix (k-anonymity)
    response = requests.get(
        f'https://api.pwnedpasswords.com/range/{prefix}'
    )
    
    if response.status_code != 200:
        return False
    
    # Check if our hash suffix appears in results
    for line in response.text.splitlines():
        hash_suffix, count = line.split(':')
        if hash_suffix == suffix:
            return True
    
    return False

def audit_compromised(vault_file: str):
    """Check all passwords against breach database."""
    with open(vault_file, 'r') as f:
        vault = json.load(f)
    
    compromised = []
    
    for item in vault.get('items', []):
        if item.get('type') != 1:
            continue
        
        password = item.get('login', {}).get('password', '')
        if password and check_compromised(password):
            compromised.append(item.get('name'))
    
    if compromised:
        print("\nCompromised passwords found in:")
        for name in compromised:
            print(f"  - {name}")
    else:
        print("\nNo compromised passwords found.")
```

Use this carefully—while the k-anonymity model protects your passwords, you're still sending partial hashes to an external service. For maximum security, consider running this check only on an offline machine.

## Creating an Audit Workflow

Rather than performing manual audits, establish a regular workflow using your password manager's built-in features combined with custom scripts.

### Automated Weekly Audit

```bash
#!/bin/bash
# Weekly vault audit script

DATE=$(date +%Y-%m-%d)
AUDIT_DIR=~/vault-audits
mkdir -p "$AUDIT_DIR"

# Export vault
bw unlock --raw
export BW_SESSION
bw export --format json --output "$AUDIT_DIR/vault-$DATE.json"

# Run analysis
python3 audit_vault.py "$AUDIT_DIR/vault-$DATE.json" > "$AUDIT_DIR/report-$DATE.txt"

# Check for compromised passwords
python3 audit_compromised.py "$AUDIT_DIR/vault-$DATE.json" >> "$AUDIT_DIR/report-$DATE.txt"

echo "Audit complete. Report saved to $AUDIT_DIR/report-$DATE.txt"
```

Schedule this with cron to run weekly:

```bash
# Add to crontab
0 9 * * 0 /path/to/weekly-audit.sh
```

## Additional Audit Areas

Beyond passwords, review these vault components:

**Two-Factor Authentication**: Check which accounts still rely on less secure 2FA methods like SMS. Identify accounts without any two-factor enabled.

**Secure Notes**: Audit sensitive information stored in secure notes. Ensure nothing critical is stored without protection.

**Shared Items**: Review items shared with family or team members. Remove access for people who no longer need it.

**Recovery Options**: Verify your recovery email and phone number are current. Check that backup codes are stored securely.

**Attachments**: Review large files stored in your vault. Consider moving these to dedicated encrypted storage solutions.

## Related Reading

- [Best Password Manager for Developers: A Technical Guide](/privacy-tools-guide/best-password-manager-for-developers/)
- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [Password Rotation Policy Best Practices: A Developer's Guide.](/privacy-tools-guide/password-rotation-policy-best-practices-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
