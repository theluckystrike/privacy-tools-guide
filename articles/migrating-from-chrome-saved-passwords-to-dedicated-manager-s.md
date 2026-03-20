---

layout: default
title: "Migrating from Chrome Saved Passwords to Dedicated."
description: "A practical guide for developers and power users moving Chrome passwords to a dedicated password manager. Export, import, and secure your credentials."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /migrating-from-chrome-saved-passwords-to-dedicated-manager-s/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
voice-checked: false
---

{% raw %}

To migrate Chrome passwords to a dedicated manager like Bitwarden or 1Password, export your Chrome passwords from chrome://settings/passwords, then import the CSV file into your new manager. This process protects your credentials from Google's access and enables advanced features like vault sharing, CLI automation, and self-hosting that Chrome's basic manager cannot provide. This guide provides step-by-step instructions for a complete, secure migration.

## Why Migrate to a Dedicated Password Manager

Chrome stores passwords with your Google account encryption, but the management interface remains limited. Dedicated managers like Bitwarden, 1Password, or Vaultwarden offer end-to-end encryption where only you hold the decryption key. These tools provide vault sharing, emergency access, CLI automation, and integration with password getters and secret injection systems.

The fundamental difference lies in who controls your keys. Chrome's "smart lock" feature syncs passwords to Google's servers, meaning Google can access your encrypted data. Self-hosted options like Bitwarden or Vaultwarden give you complete ownership.

## Step 1: Export Chrome Passwords

Before importing elsewhere, you need to extract your passwords from Chrome. Several methods exist, ranging from built-in exports to direct database access.

### Using Chrome's Built-in Export

Navigate to `chrome://settings/passwords` in your browser. Click the three-dot menu next to "Saved Passwords" and select "Export passwords." Chrome will prompt for your computer password and download a CSV file.

### Direct Database Extraction (More Complete)

Chrome stores passwords in a SQLite database. On macOS, access it directly for a more complete export:

```bash
# Find Chrome's Login Data file
find ~/Library/Application\ Support/Google/Chrome -name "Login Data" -type f

# Copy to temp location (Chrome locks the original)
cp ~/Library/Application\ Support/Google/Chrome/Default/Login\ Data /tmp/chrome_logins

# Query the database with sqlite3
sqlite3 /tmp/chrome_logins "SELECT origin_url, username_value, password_value FROM logins"
```

The password values are encrypted with DPAPI on macOS/Windows. For plaintext extraction, use a tool like `chrome-decrypt` or the `hashcat` approach if you need to crack your own exported data.

### Recommended: Use a Python Script for Clean Export

The `chrome-export` tool provides cleaner CSV output:

```bash
# Install chrome-export
pip install chrome-export

# Export to CSV
chrome-export --csv /tmp/chrome_passwords.csv
```

This script handles the decryption automatically using your local Chrome keychain.

## Step 2: Choose Your Destination Manager

Select a password manager that fits your threat model and technical requirements:

- **Bitwarden**: Open-source, cloud-hosted or self-hosted, excellent CLI
- **1Password**: Closed-source, best-in-class UX, robust CLI
- **Vaultwarden**: Self-hosted Bitwarden alternative, lightweight
- **KeepassXC**: Local-only, completely offline, excellent for air-gapped systems

For developers, Bitwarden's CLI (`bw`) offers excellent scripting capabilities. Install it:

```bash
# macOS
brew install bitwarden-cli

# Linux (Debian/Ubuntu)
sudo dpkg -i bitwarden-cli_*.deb

# Verify installation
bw --version
```

## Step 3: Import to Your Dedicated Manager

### Bitwarden Import Process

First, authenticate with Bitwarden:

```bash
bw login your@email.com
# Enter master password when prompted

# Unlock your vault
bw unlock
# Copy the session key shown

# Set session for subsequent commands
export BW_SESSION="your_session_key_here"
```

Bitwarden accepts CSV imports with specific formatting. Your Chrome export may need reformatting:

```bash
# Reformat CSV for Bitwarden import
# Required columns: folder,type,login_uri,login_username,login_password,login_note

# Import the formatted CSV
bw import bitwardencsv /path/to/your/formatted_passwords.csv
```

### 1Password Import Process

1Password requires CSV transformation using their template:

```bash
# Download 1Password CLI
brew install --cask 1password-cli

# Sign in
op signin

# Create items from CSV (requires specific format)
# 1Password provides import templates on their website
```

### Manual Import with CLI (Most Control)

For full control, create entries programmatically:

```bash
# Bitwarden: Create individual items
bw get item "New Item" 2>/dev/null || \
bw create item "{
  \"name\": \"Example Site\",
  \"login\": {
    \"username\": \"user@example.com\",
    \"password\": \"your_password_here\",
    \"uri\": \"https://example.com\"
  }
}"
```

## Step 4: Verify and Clean Up

After import, verify your data transferred correctly:

```bash
# Bitwarden: List all items
bw list items | jq '.[] | .name'

# Check item count matches expectations
bw list items | jq 'length'
```

Review entries for duplicates or errors. Chrome often creates duplicate entries for the same site with slight URL variations.

## Step 5: Disable Chrome Password Saving

Prevent future passwords from accumulating in Chrome:

1. Open Chrome settings
2. Navigate to "Passwords"
3. Toggle off "Offer to save passwords"
4. Toggle off "Auto Sign-in"

For enterprise environments, Chrome policies can enforce these settings:

```json
// chrome_policy.json
{
  "PasswordManagerEnabled": false,
  "AutoSignInEnabled": false
}
```

Deploy via MDM or group policy.

## Step 6: Set Up CLI Automation (Recommended)

Now that you've migrated, use CLI access for development workflows:

```bash
# Bitwarden: Get password programmatically
bw get password "github.com" --raw

# Bitwarden: Get username
bw get username "github.com" --raw

# Bitwarden: Generate TOTP code
bw get totp "github.com" --raw
```

Integrate into your shell profile:

```bash
# Add to ~/.zshrc or ~/.bashrc
export BW_SESSION="$(bw unlock --raw)"

# Function to get secrets easily
get_secret() {
  bw get "$1" "$2" --"$3"
}
```

Use these for environment variable injection in deployment scripts:

```bash
# Inject into environment
export GITHUB_TOKEN="$(bw get password "github.com" --field password)"
export DB_PASSWORD="$(bw get password "production-db" --field password)"
```

## Troubleshooting Common Issues

### CSV Formatting Errors

Import failures often stem from encoding or format issues:

```bash
# Convert to UTF-8
iconv -f ISO-8859-1 -t UTF-8 input.csv > output.csv

# Remove BOM if present
sed -i '' '1s/^\xEF\xBB\xBF//' output.csv
```

### Master Password Complexity

Dedicated managers enforce stronger password policies. Ensure your master password meets requirements and consider using a passphrase:

```bash
# Generate a secure master password
openssl rand -base64 24
```

### Session Timeout

CLI sessions expire for security. Automate re-authentication:

```bash
# Refresh session automatically
export BW_SESSION="$(bw unlock --raw 2>/dev/null)"
```

## Security Considerations

After migration, enhance your security posture:

1. Enable two-factor authentication on your password manager account
2. Set up emergency access for trusted contacts
3. Use unique, generated passwords for each site (your manager handles remembering them)
4. Enable vault timeout (auto-lock after inactivity)

## Conclusion

Migrating from Chrome's built-in manager to a dedicated solution gives you programmatic access, stronger encryption, and complete ownership of your credentials. The export and import processes require some command-line work, but the long-term benefits for developers and power users justify the initial effort.

Start with a small batch of critical passwords, validate your workflow, then migrate the remainder. Your future self will appreciate having CLI access to secrets when automating deployments or managing infrastructure.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
