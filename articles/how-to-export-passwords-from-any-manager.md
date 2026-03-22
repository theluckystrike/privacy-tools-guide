---
layout: default
title: "How to Export Passwords from Any Manager"
description: "To export passwords from any manager, use its CLI tool for the most control: op vault export for 1Password, bw export --format json for Bitwarden, lpass export"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /how-to-export-passwords-from-any-manager/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---
---
layout: default
title: "How to Export Passwords from Any Manager"
description: "To export passwords from any manager, use its CLI tool for the most control: op vault export for 1Password, bw export --format json for Bitwarden, lpass export"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /how-to-export-passwords-from-any-manager/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

To export passwords from any manager, use its CLI tool for the most control: `op vault export` for 1Password, `bw export --format json` for Bitwarden, `lpass export` for LastPass, `keepassxc-cli export` for KeePassXC, or `dcli export` for Dashlane. Each outputs CSV or JSON that you should encrypt immediately with GPG and delete the plaintext file. This guide covers step-by-step export instructions for all major password managers, including web interface alternatives and automated backup scripts.

## Key Takeaways

- **Switching to a different password manager is the most common**: you might prefer an open-source solution or need features unavailable in your current tool.
- **The platform supports CSV**: export containing usernames, passwords, URLs, and notes.
- **The CSV option provides**: broader compatibility, while 1PIF preserves more metadata.
- **You can choose between**: CSV and JSON formats.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Consider a security review**: if your application handles sensitive user data.

## Why Export Your Passwords?

Several legitimate scenarios require password export. Switching to a different password manager is the most common—you might prefer an open-source solution or need features unavailable in your current tool. Creating regular encrypted backups provides insurance against account lockouts or service outages. Developers building integration tools need programmatic access to credential data. Security researchers auditing password practices may need to analyze stored credentials.

Whatever your reason, the export process should preserve your data in an usable format while maintaining security during transfer.

## 1Password Export Methods

1Password offers several export pathways depending on your plan and preferences.

### Using the 1Password CLI

The 1Password CLI provides the most flexible export capability. First, ensure you have the CLI installed and authenticated:

```bash
# Install on macOS
brew install --cask 1password-cli

# Authenticate with your account
op signin
```

Export your vault items to JSON format:

```bash
# Export all items to JSON
op vault export "Personal" --format json > passwords.json

# Export specific item types
op item list --vault "Personal" --format json | jq -r '.[].id' | while read id; do
  op item get "$id" --format json
done > export.json
```

The JSON output includes item UUIDs, titles, usernames, passwords, URLs, and custom fields. You can parse this data with standard tools like `jq` for further processing.

### Exporting from the 1Password Web App

Navigate to your vault in the web interface, select items, and choose "Export" from the actions menu. You can select CSV or 1PIF (1Password Interchange Format) as output formats. The CSV option provides broader compatibility, while 1PIF preserves more metadata.

## Bitwarden Export Options

Bitwarden's open-source nature makes export straightforward.

### Web Vault Export

In the web vault, access Settings > Export Data. You can choose between CSV and JSON formats. JSON preserves more detail including custom fields, totp secrets, and folder associations.

### Using the Bitwarden CLI

The Bitwarden CLI offers programmatic export:

```bash
# Install Bitwarden CLI
npm install -g @bitwarden/cli

# Login interactively
bw login

# Unlock your vault
bw unlock

# Export to JSON
bw export --format json --output passwords.json
```

For automated scripts, you can use an API key:

```bash
BW_CLIENT_ID="your-client-id" BW_CLIENT_SECRET="your-secret" bw login --api

# Then export with session
bw unlock --passwordenv BW_PASSWORD
```

The exported JSON includes login items, secure notes, identities, and cards—each with full field data.

## LastPass Export Procedures

LastPass provides export functionality through its web interface and CLI tools.

### Web Interface Export

LastPass offers export under Account Settings > Export. The platform supports CSV export containing usernames, passwords, URLs, and notes. Navigate to the Advanced options section to access this functionality.

### Using the LastPass CLI (lpass)

The LastPass CLI provides more control:

```bash
# Install on macOS
brew install lastpass-cli

# Export all entries
lpass export > passwords.csv

# Export to JSON with better structure
lpass export --json > passwords.json
```

Note that LastPass requires your master password for export. Some enterprise accounts may have export disabled by administrators.

## KeePass and KeePassXC Export

As open-source, local password managers, KeePass variants offer the most direct access to your data.

### Direct Database Export

KeePass stores passwords in `.kdbx` files. The format is encrypted, but you can export to multiple formats:

```bash
# Using keepassxc-cli (part of KeePassXC)
keepassxc-cli export --format csv /path/to/database.kdbx passwords.csv

# Usingkpcli for scripting
kpcli --kdb=passwords.kdbx --export=passwords.csv
```

### Python Script for KeePass Export

You can use the `pykeepass` library for programmatic access:

```python
from pykeepass import PyKeePass

# Open your KeePass database
kp = PyKeePass('passwords.kdbx', password='your-master-password')

# Export all entries
entries_data = []
for entry in kp.entries:
    entries_data.append({
        'title': entry.title,
        'username': entry.username,
        'password': entry.password,
        'url': entry.url,
        'notes': entry.notes
    })

import json
with open('export.json', 'w') as f:
    json.dump(entries_data, f, indent=2)
```

This approach gives you complete control over the export format and allows filtering or transformation before saving.

## Dashlane Export

Dashlane provides export through its web interface and desktop applications.

### Web Interface Export

Access your Dashlane account through the web, navigate to Settings > Passwords > Export. You can choose between CSV and JSON formats. The export includes passwords, personal info, and payment methods depending on your selection.

### Using the Dashlane CLI

Dashlane's CLI tool supports export:

```bash
# Install dashlane-cli
npm install -g dashlane-cli

# Authenticate
dcli login

# Export passwords
dcli export --format json > passwords.json
```

## Security Considerations

Regardless of which manager you export from, handle the exported file carefully:

Encrypt the exported file immediately—convert CSV output to GPG or a password-protected ZIP before storing it anywhere. Remove the plaintext file from disk after transferring to secure storage, and never send an unencrypted password export over an unencrypted channel. After migrating, generate new passwords for critical accounts and confirm that the export contains all expected entries before deleting source data.

## Automating Regular Backups

For developers wanting automated exports, create a scheduled script:

```bash
#!/bin/bash
# backup-passwords.sh

DATE=$(date +%Y%m%d)
BACKUP_DIR="$HOME/password-backups"

# Create backup directory if needed
mkdir -p "$BACKUP_DIR"

# Export Bitwarden (example)
export BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)
bw export --format json --file "backup-$DATE.json" --output "$BACKUP_DIR"

# Encrypt the backup
gpg -c "$BACKUP_DIR/backup-$DATE.json"
rm "$BACKUP_DIR/backup-$DATE.json"

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/*.gpg | tail -n +8 | xargs rm
```

Schedule this with cron for regular automated backups:

```bash
# Add to crontab - run daily at 2 AM
0 2 * * * /path/to/backup-passwords.sh
```

Most modern password managers support CLI export, which gives the most control over output format and automation. Encrypt the file immediately after export and verify it contains all expected entries before removing source data.

## Frequently Asked Questions

**How long does it take to export passwords from any manager?**

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

- [Migrating From Chrome Saved Passwords To Dedicated Manager S](/privacy-tools-guide/migrating-from-chrome-saved-passwords-to-dedicated-manager-s/)
- [Does Mullvad Work in Turkmenistan? 2026 Technical Analysis](/privacy-tools-guide/does-mullvad-work-in-turkmenistan-2026-any-server-works/)
- [How to Use Public Computers Safely Without Leaving Any Trace](/privacy-tools-guide/how-to-use-public-computers-safely-without-leaving-any-trace/)
- [How To Share Passwords Securely With Team Using Encrypted Co](/privacy-tools-guide/how-to-share-passwords-securely-with-team-using-encrypted-co/)
- [Passkeys vs Passwords: Security Comparison FIDO2 WebAuthn](/privacy-tools-guide/passkeys-vs-passwords-security-comparison-fido2-webauthn-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
