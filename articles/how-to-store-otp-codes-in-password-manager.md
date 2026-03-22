---
layout: default
title: "How To Store Otp Codes In Password Manager"
description: "Managing multiple authenticator apps across devices creates friction. Developers and power users often prefer consolidating OTP (One-Time Password) codes"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /how-to-store-otp-codes-in-password-manager/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---
---
layout: default
title: "How To Store Otp Codes In Password Manager"
description: "Managing multiple authenticator apps across devices creates friction. Developers and power users often prefer consolidating OTP (One-Time Password) codes"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /how-to-store-otp-codes-in-password-manager/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

Managing multiple authenticator apps across devices creates friction. Developers and power users often prefer consolidating OTP (One-Time Password) codes within their password manager for unified access, easier backups, and cross-device synchronization. This guide explores practical methods for storing TOTP codes in password managers, with emphasis on CLI tools and programmatic approaches.

## Understanding TOTP Storage Mechanisms

TOTP (Time-based One-Time Password) codes rely on a shared secret key—the base32-encoded seed that both your authenticator app and the service generate codes from. When you store this secret in your password manager, you generate the same codes any authenticator would produce.

The secret typically arrives in two formats when setting up 2FA:
- **otpauth:// URI** — an URL containing the secret and parameters
- **Manual entry** — The base32 secret string alone

Password managers like Bitwarden and 1Password capture these automatically during 2FA setup, but you can also manually add them for existing accounts.

## Storing OTP in Bitwarden

Bitwarden offers native TOTP integration. The easiest method involves using the browser extension or desktop app during login setup. When a service presents a QR code, Bitwarden's "Add Login" dialog includes an option to scan and automatically populate the TOTP field.

### Manual Secret Entry

For programmatic or manual entry, store the otpauth URI in the custom field:

```json
{
  "username": "developer@example.com",
  "password": "your_master_password",
  "totp": "otpauth://totp/Example:dev@email.com?secret=JBSWY3DPEHPK3PXP&issuer=Example"
}
```

In Bitwarden's web vault, add a custom field of type "hidden" and paste the full otpauth URI. The vault will then display the current TOTP code alongside your login.

### Using Bitwarden CLI

The Bitwarden CLI provides programmatic access:

```bash
# List all items with TOTP
bw list items --search "totp" | jq '.[] | select(.login.totp != null)'

# Get TOTP code for specific item
bw get item "github" | jq -r '.login.totp'
```

For automation scripts, unlock the vault programmatically:

```bash
export BW_SESSION=$(bw unlock --raw)
bw get item "github" --pretty | jq -r '.login.totp'
```

This enables cron jobs or scripts that fetch current codes without opening the UI.

## Storing OTP in 1Password

1Password handles TOTP similarly, embedding the secret within each login item. During 2FA setup on any website, 1Password's browser extension intercepts the QR code and automatically stores the secret.

### Using the 1Password CLI

The op CLI tool provides full access:

```bash
# Get a specific item's OTP
op item get "GitHub" --format json | jq -r '.otp'

# List items with OTP enabled
op vault get "Private" | jq -r '.items[]' | xargs -I {} op item get {} | jq -r 'select(.otp != null) | .title'
```

For developers integrating into workflows:

```bash
#!/bin/bash
# Fetch current OTP for a service
SERVICE="$1"
OTP=$(op item get "$SERVICE" --format json 2>/dev/null | jq -r '.otp')

if [ -z "$OTP" ] || [ "$OTP" = "null" ]; then
    echo "No OTP found for $SERVICE"
    exit 1
fi

echo "$OTP"
```

## KeePass and KeePassXC Approach

For users preferring open-source solutions, KeePassXC stores TOTP through plugins:

1. Enable the "KeePassXC-Browser" extension in your browser
2. Configure the TOTP plugin in KeePassXC settings
3. When adding entries, use "Add entry" → "Advanced" → "Additional Attributes"

The TOTP appears in the entry and auto-fills like standard credentials.

### CLI Automation with keepassxc-cli

```bash
# Generate TOTP for an entry
keepassxc-cli totp database.kdbx -k keyfile.key EntryName

# Export to use in scripts
export KEEPASSXC_CLI_PASS=master_password
keepassxc-cli totp database.kdbx -s EntryName
```

This approach works well for users who want offline-first storage without cloud synchronization.

## Extracting TOTP Secrets from Existing Authenticator Apps

If you already use apps like Google Authenticator or Authy, migrating to password manager storage requires extracting the secrets. Several methods exist:

### Using an Android Emulator with ADB

```bash
# Pull the authenticator database (requires root)
adb shell
su
cp /data/data/com.google.android.apps.authenticator2/databases/databases /sdcard/Download/
exit
adb pull /sdcard/Download/databases ./authenticator.db
```

The database uses SQLite. Extract secrets:

```bash
sqlite3 authenticator.db "SELECT email, secret FROM accounts;"
```

### Using Authy Export (if enabled)

Authy provides encrypted backups. Enable them in settings, then export. The backup contains all your 2FA secrets in a decryptable format.

## Security Considerations

Storing TOTP secrets in your password manager consolidates attack surface. Consider these tradeoffs:

**Advantages:**
- Single master password protects all credentials
- Encrypted vault syncs across devices
- Easier backup and recovery
- No phone dependency for code access

**Risks:**
- If master password is compromised, attackers gain both passwords and 2FA
- Some password managers transmit TOTP codes to servers for sync
- Loss of master password means loss of all 2FA

For high-security scenarios, maintain separate storage:
- Password manager for credentials
- Dedicated hardware token (YubiKey) for critical 2FA
- Paper backup of recovery codes in secure location

## Automating TOTP in Development Workflows

Developers often need TOTP codes in CI/CD pipelines or deployment scripts:

```python
#!/usr/bin/env python3
import subprocess
import json
import sys
import time

def get_bitwarden_totp(item_name: str) -> str:
    """Fetch TOTP from Bitwarden vault."""
    result = subprocess.run(
        ['bw', 'get', 'item', item_name],
        capture_output=True,
        text=True,
        env={**subprocess.os.environ, 'BW_SESSION': subprocess.run(
            ['bw', 'unlock', '--raw'],
            capture_output=True,
            text=True
        ).stdout.strip()}
    )

    data = json.loads(result.stdout)
    return data['login']['totp']

# Usage in deployment script
if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: fetch_totp.py <service>")
        sys.exit(1)

    print(get_bitwarden_totp(sys.argv[1]))
```

This pattern integrates with deployment tools that require 2FA-enabled accounts.

## When to Avoid Password Manager TOTP

Some scenarios warrant keeping authenticator apps separate:

- **High-value targets** (financial institutions, cryptocurrency exchanges)
- **Shared accounts** where multiple people need 2FA access
- **Organizations requiring dedicated authenticator hardware**

Hardware security keys (YubiKey, SoloKey) provide stronger protection than TOTP codes stored in software.

## Frequently Asked Questions

**How long does it take to store otp codes in password manager?**

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

- [Best Password Manager CLI Tools: A Developer's Guide](/privacy-tools-guide/best-password-manager-cli-tools/)
- [Best Password Manager for Android 2026: A Developer's Guide](/privacy-tools-guide/best-password-manager-for-android-2026/)
- [Best Password Manager for Developers: A Technical Guide](/privacy-tools-guide/best-password-manager-for-developers/)
- [Best Password Manager for Enterprise: A Technical Guide](/privacy-tools-guide/best-password-manager-for-enterprise/)
- [Best Password Manager For Firefox Extension](/privacy-tools-guide/best-password-manager-for-firefox-extension/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
