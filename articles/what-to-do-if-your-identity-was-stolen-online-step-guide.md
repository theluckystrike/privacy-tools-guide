---

layout: default
title: "What to Do If Your Identity Was Stolen Online: Step-by-Step Guide"
description: "A practical guide for developers and power users on how to respond when your identity has been compromised online. Includes command-line tools, API checks, and automation scripts."
date: 2026-03-16
author: theluckystrike
permalink: /what-to-do-if-your-identity-was-stolen-online-step-guide/
categories: [security, guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Discovering that your identity has been stolen online can be alarming. Whether it's a compromised email account, stolen API keys, or fraudulent use of your personal information, acting quickly and systematically minimizes damage. This guide provides a technical, step-by-step approach tailored for developers and power users who want more than generic advice.

## Step 1: Audit Your Digital Footprint

Before taking action, understand the scope of the breach. List all accounts, services, and tokens that might be affected.

### Using Have I Been Pwned for Email Breaches

The Have I Been Pwned API lets you check if your email appears in known data breaches:

```bash
# Check if your email appears in breaches
curl -H "hibp-api-key: YOUR_API_KEY" \
  "https://haveibeenpwned.com/api/v3/breachedaccount/your@email.com"
```

For automated monitoring, you can query the service programmatically and integrate it into your alerting systems.

### Checking Exposed Passwords

Use the k-anonymity model to check if your passwords have been exposed without sending your actual password over the network:

```bash
# Hash your password with SHA-1 and check the first 5 characters
echo -n "yourpassword" | sha1 | tr '[:upper:]' '[:lower:]'
# Take the first 5 characters and query the API
curl "https://api.pwnedpasswords.com/range/ABCDE"
```

This returns a list of suffixes. If any match the remainder of your hash, your password has been compromised.

## Step 2: Secure Compromised Accounts Immediately

### Reset Passwords with a Password Manager

Generate strong, unique passwords for each affected account. If you use 1Password or Bitwarden, you can automate this:

```bash
# Using Bitwarden CLI to generate a secure password
bw generate --length 24 --includeNumber --includeSymbol --includeLower --includeUpper
```

### Revoke Active Sessions

Many services support session revocation through their security settings or APIs. For Google accounts:

1. Go to Security > Manage all your devices
2. Sign out of all unknown devices
3. Enable 2FA if not already active

For GitHub, review and revoke suspicious OAuth applications and personal access tokens in Settings > Applications.

## Step 3: Rotate Compromised Credentials

If you're a developer, API keys and tokens are high-value targets. Rotate anything that may have been exposed.

### Rotating GitHub Personal Access Tokens

```bash
# List your active tokens (requires authentication)
gh auth token
gh api --method GET /user/authorizations
```

Create a new token with minimal scopes needed for your workflows, then update your CI/CD pipelines and environment variables.

### Revoking and Regenerating API Keys

For AWS, rotate access keys immediately:

```bash
# List access keys
aws iam list-access-keys --user-name YOUR_USERNAME

# Create a new key
aws iam create-access-key --user-name YOUR_USERNAME

# Delete the old key (use the key ID from list-access-keys)
aws iam delete-access-key --user-name YOUR_USERNAME --access-key-id AKIAIOSFODNN7EXAMPLE
```

Similar patterns apply to Google Cloud, Azure, and other cloud providers.

## Step 4: Enable Strong Authentication Everywhere

Passwords alone are insufficient. Enable multi-factor authentication (MFA) on all critical accounts.

### Hardware Security Keys

For maximum security, use a hardware key like YubiKey:

```bash
# Verify YubiKey is detected (on Linux)
lsusb | grep Yubico
```

Register your hardware key with services that support FIDO2/WebAuthn, including Google, GitHub, and cloud providers.

### Time-Based One-Time Passwords (TOTP)

If hardware keys aren't available, use TOTP apps like Authy or Aegis. Avoid SMS-based 2FA whenever possible due to SIM swapping vulnerabilities.

## Step 5: Monitor for Unauthorized Activity

Set up alerts for suspicious behavior across your accounts.

### Setting Up Account Activity Alerts

For GitHub, enable security alerts in Settings > Notifications. For cloud accounts, use services like AWS GuardDuty or Azure Security Center.

### Using Have I Been Pwned for Continuous Monitoring

Register your email for breach notifications:

```bash
# Send a POST request to subscribe to notifications
curl -X POST "https://haveibeenpwned.com/api/v3/breachalert" \
  -H "hibp-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "your@email.com"}'
```

## Step 6: Document Everything

Keep records of the incident for potential legal or insurance purposes.

### Creating an Incident Log

```markdown
# Identity Theft Incident Log

Date discovered: 2026-03-16
Accounts affected: [list]
Actions taken: [list with dates]
Police report filed: [yes/no, reference number]
```

Include screenshots, email correspondence, and any communications with service providers.

## Step 7: Consider Identity Protection Services

While this guide focuses on technical steps, you may want to freeze your credit reports with the major bureaus (Equifax, Experian, TransUnion) to prevent fraudulent accounts from being opened in your name.

## Prevention for the Future

After recovering from an identity theft incident, implement these ongoing practices:

- Use a password manager to generate and store unique passwords
- Enable MFA on every account that supports it
- Regularly audit third-party app permissions
- Rotate API keys and tokens on a schedule
- Use a separate email alias for different services to isolate breaches

## Summary

Responding to identity theft requires speed and method. Audit your exposure, secure accounts by resetting passwords and enabling MFA, rotate credentials, monitor for further abuse, document everything, and consider credit freezes. With these steps, you regain control and reduce the risk of future incidents.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
