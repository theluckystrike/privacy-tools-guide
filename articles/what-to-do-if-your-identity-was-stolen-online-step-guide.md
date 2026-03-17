---
layout: default
title: "What to Do If Your Identity Was Stolen Online - Step-by-Step Recovery Guide"
description: "A comprehensive guide for developers and power users on how to recover from identity theft, including immediate actions, security hardening, and monitoring tools."
date: 2026-03-16
author: theluckystrike
permalink: /what-to-do-if-your-identity-was-stolen-online-step-guide/
---

Identity theft is one of the most distressing digital security incidents you can experience. When someone steals your personal information—your name, Social Security number, bank credentials, or email accounts—they can open new lines of credit, make unauthorized purchases, or commit crimes under your name. For developers and power users who maintain extensive digital footprints, the risk is particularly high.

This guide walks you through the exact steps to take when you discover your identity has been compromised, with specific technical recommendations for securing your accounts and preventing future incidents.

## Step 1: Contain the Damage Immediately

The moment you suspect identity theft, your first priority is limiting what the attacker can do with your stolen information.

### Lock or Freeze Your Credit

If your Social Security number or financial information was exposed, immediately place a credit freeze with the three major bureaus:

```bash
# Equifax
# https://www.equifax.com/personal/credit-report-services/credit-freeze/

# Experian  
# https://www.experian.com/freeze/center.html

# TransUnion
# https://www.transunion.com/credit-freeze
```

A credit freeze prevents new accounts from being opened in your name. This is stronger than a fraud alert, which only requires creditors to verify your identity before extending credit.

### Revoke Session Tokens and API Keys

For developers, identity theft often means compromised API keys, OAuth tokens, or session credentials. Immediately revoke all active sessions:

```javascript
// Example: Revoking all OAuth tokens via provider APIs
// GitHub
await fetch('https://api.github.com/user/authorizations', {
  method: 'DELETE',
  headers: { 'Authorization': `token ${your_token}` }
});

// Google
// Visit: https://myaccount.google.com/permissions
// Revoke access for all unfamiliar applications

// For AWS IAM users:
aws iam list-access-keys --user-name YourUserName
aws iam update-access-key --access-key-id AKIA... --status Inactive --user-name YourUserName
```

### Change All Passwords

Generate new, strong passwords for all critical accounts. Use a password manager to create unique passwords:

```bash
# Using openssl to generate a secure random password
openssl rand -base64 20
```

Prioritize in this order:
1. Email accounts (especially your primary email)
2. Financial institutions
3. Password manager master password
4. Social media accounts

## Step 2: Document Everything

Create a detailed log of the incident for law enforcement and your own reference.

### What to Document

- Date and time you discovered the theft
- What information was compromised
- Any suspicious activities you noticed
- Communications with attackers (if any)
- Steps you've taken to mitigate the damage

### File a Report

File reports with the following organizations:

1. **Federal Trade Commission (FTC)**: Visit [identitytheft.gov](https://identitytheft.gov) to create an identity theft affidavit
2. **Local police department**: Get a police report number
3. **FBI Internet Crime Complaint Center (IC3)**: [ic3.gov](https://ic3.gov) for online crimes

```yaml
# Example identity theft affidavit structure
incidents:
  - date_discovered: "2026-03-16"
    type: "identity_theft"
    compromised_data:
      - "Email address"
      - "Social Security Number"
      - "Bank account credentials"
    actions_taken:
      - "Placed credit freeze"
      - "Changed all passwords"
      - "Filed FTC report"
    police_report_number: "XX-XXXXX"
```

## Step 3: Secure Your Digital Presence

For developers and power users, your digital identity extends beyond traditional accounts.

### Rotate API Keys and Secrets

Review all API keys, OAuth tokens, and application secrets. Assume that any credential stored alongside compromised data is also potentially exposed.

```python
# Example: Auditing API keys in your codebase
import subprocess
import re

def find_potential_keys(filepath):
    patterns = [
        r'ghp_[a-zA-Z0-9]{36}',  # GitHub tokens
        r'AKIA[0-9A-Z]{16}',       # AWS access keys
        r'sk-[a-zA-Z0-9]{48}',     # OpenAI API keys
    ]
    
    with open(filepath, 'r') as f:
        for line_num, line in enumerate(f, 1):
            for pattern in patterns:
                if re.search(pattern, line):
                    print(f"Line {line_num}: Potential key found")

# Run against your application files
find_potential_keys('config/secrets.py')
```

### Enable Two-Factor Authentication Everywhere

Switch from SMS-based 2FA to more secure methods:

- Hardware security keys (YubiKey, Titan)
- TOTP authenticator apps (Authy, Bitwarden Authenticator)
- Passkeys where supported

```bash
# Verify 2FA status across your accounts
# Use services like HaveIBeenPwned to check compromised accounts
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/youremail@example.com"
```

### Audit Connected Applications

Review third-party applications that have access to your accounts. Remove any you don't recognize or no longer use.

## Step 4: Monitor and Recover

### Set Up Identity Monitoring

Consider these monitoring services:

- **Credit monitoring**: Credit Karma, Experian Free Credit Score
- **Dark web monitoring**: HaveIBeenPwned, BreachDirectory
- **Identity theft protection**: LifeLock, IdentityForce

```bash
# Check if your email appears in new breaches
# Using HIBP API (rate-limited)
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/youremail@example.com?truncateResponse=false" \
  -H "User-Agent: IdentityMonitoring"
```

### Review Account Logs

For each compromised account, review login history and account activity:

```javascript
// Example: Checking recent GitHub account activity
// Visit: https://github.com/settings/activity

// Look for:
// - Unknown IP addresses
// - Unfamiliar repositories created
// - Unauthorized OAuth grants
// - Unexpected settings changes
```

### Check for Medical Identity Theft

Review your insurance Explanation of Benefits statements. Medical identity theft can result in fraudulent claims being attached to your health record.

## Step 5: Prevent Future Incidents

### Implement Defense in Depth

- Use a password manager (Bitwarden, 1Password, or self-hosted Vault)
- Enable hardware 2FA for critical accounts
- Use separate emails for different account types
- Run regular security audits on your digital presence

```bash
# Example: Security audit script for personal accounts
#!/bin/bash
# security-audit.sh

echo "=== Security Audit ==="
echo ""
echo "Checking password strength..."
pwscore  # Install: pip install pwscore

echo ""
echo "Checking for compromised passwords..."
curl -s "https://api.pwnedpasswords.com/range/$(echo -n 'yourpassword' | sha1 | cut -c1-5)"

echo ""
echo "Review connected devices..."
# Check your Google account
# Visit: https://myaccount.google.com/device-activity
```

### Set Up Alerts

Configure alerts for:
- New credit inquiries
- Account login from new devices
- Changes to critical account settings
- Dark web exposure of your information

## Conclusion

Identity theft recovery is a marathon, not a sprint. The steps you take in the first 72 hours are critical, but ongoing vigilance is essential. By documenting everything, securing your digital presence, and implementing robust monitoring, you can minimize the damage and protect yourself from future incidents.

Remember: the goal isn't just recovery—it's building a more resilient digital identity that can withstand future threats.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
