---
---
layout: default
title: "Secure Email Forwarding With Encryption How To Set Up"
description: "A practical guide to setting up secure email forwarding with encryption using Anonaddy and SimpleLogin. Learn how to protect your inbox with alias"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /secure-email-forwarding-with-encryption-how-to-set-up-anonad/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

Email privacy is a fundamental concern for developers and power users who manage multiple online accounts. Exposing your primary email address leads to spam, tracking, and potential data breaches. This guide covers setting up secure email forwarding with alias services Anonaddy and SimpleLogin, combined with PGP encryption to protect your forwarded emails end-to-end.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced: API Integration for Developers](#advanced-api-integration-for-developers)
- [Best Practices for Production Use](#best-practices-for-production-use)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Email Alias Services

Email alias services act as intermediaries between your real inbox and the outside world. Instead of giving out your primary email, you create unique aliases that forward to your actual inbox. When an alias gets compromised or sold to spammers, you simply delete it—your primary address remains untouched.

Both Anonaddy and SimpleLogin offer browser extensions, mobile apps, and SMTP forwarding capabilities. The key difference lies in their architecture: Anonaddy is open-source and self-hostable, while SimpleLogin offers both hosted and self-hosted versions.

### Step 2: Set Up Anonaddy

Anonaddy provides a free tier with limited features and paid plans for additional aliases and domains. The self-hosted version requires Docker and a mail server, but we'll focus on the hosted version for quick setup.

### Creating Your First Alias

After creating an account at anonaddy.com, navigate to the dashboard and click "Create New Alias." You can choose between a random alias (`random@anonaddy.com`) or a custom format using your own domain.

For developers wanting programmatic control, Anonaddy offers a REST API:

```bash
# Create an alias via API
curl -X POST "https://anonaddy.com/api/v1/aliases" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"alias": {"email": "github-alias@yourdomain.com"}}'
```

### Configuring PGP Encryption

Anonaddy supports automatic PGP encryption for forwarded emails. To enable this:

1. Navigate to Settings → PGP Keys
2. Generate a new key pair or import your existing one
3. Toggle "Encrypt forwarded emails" to on

Now all forwarded emails will be encrypted with your public key before delivery. You'll need your private key to decrypt and read them.

Generate a new key pair if needed:

```bash
gpg --full-generate-key
# Choose RSA (4096 bits), set expiration, and add your identity

# Export your public key for import into Anonaddy
gpg --armor --export your@email.com > public_key.asc
```

### Step 3: Set Up SimpleLogin

SimpleLogin provides a similar workflow with a polished interface. The free tier includes unlimited aliases on the `@simplelogin.com` domain, while premium plans support custom domains.

### Initial Configuration

Create an account at simplelogin.com and install the browser extension for one-click alias generation on any email input field. The extension intercepts form submissions and automatically creates aliases.

For command-line enthusiasts, SimpleLogin offers a CLI tool:

```bash
# Install SimpleLogin CLI
pip install simplelogin-cli

# Initialize with your API key
simplelogin init YOUR_API_KEY

# Create a new alias
simplelogin alias create --prefix "developer"
```

### Self-Hosting SimpleLogin

For privacy-conscious organizations, SimpleLogin's self-hosted version runs on Docker:

```yaml
# docker-compose.yml
version: '3'
services:
  simplelogin:
    image: simplelogin/frontend:latest
    ports:
      - "3000:3000"
    environment:
      - DOMAIN=yourdomain.com
      - DATABASE_URL=postgresql://user:pass@db:5432/simplelogin
    depends_on:
      - db
      - redis
```

### Step 4: Implementing End-to-End Encryption

Both services forward emails as plain text by default. For sensitive communications, implement PGP encryption:

### Automatic Encryption with Procmail

If you're self-hosting or using a custom mail server, route incoming mail through procmail for automatic encryption:

```bash
# .procmailrc
:0
* ^To:.*
{
  # Forward to encrypted mailbox
  :0 c
  ! encrypted-forward@archive.com

  # Keep copy with PGP encryption
  :0 w
  | gpg --encrypt --recipient your@email.com --output ~/mail/encrypted.mbox
}
```

### Using Maildir with Encryption

Store encrypted emails locally using Maildir format:

```bash
# Create encrypted Maildir
mkdir -p ~/mail/.encrypted/{new,cur,tmp}

# Configure fetchmail for encrypted delivery
poll imap.yourprovider.com
  protocol IMAP
  user "your@email.com"
  password "app-specific-password"
  mda "gpg --encrypt --recipient your@email.com --output ~/mail/.encrypted/new/$(date +%s).eml"
```

## Advanced: API Integration for Developers

Both services expose APIs for programmatic alias management:

### Anonaddy API Example

```python
import requests

class EmailAliasManager:
    def __init__(self, api_token, base_url="https://anonaddy.com"):
        self.api_token = api_token
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_token}",
            "Content-Type": "application/json"
        }

    def create_alias(self, domain="@anonaddy.com", prefix=None):
        endpoint = f"{self.base_url}/api/v1/aliases"
        data = {
            "alias": {
                "domain": domain,
                "local_part": prefix
            }
        }
        response = requests.post(endpoint, json=data, headers=self.headers)
        return response.json()

    def list_aliases(self):
        endpoint = f"{self.base_url}/api/v1/aliases"
        response = requests.get(endpoint, headers=self.headers)
        return response.json()
```

### SimpleLogin API Example

```python
import requests

class SimpleLoginClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://app.simplelogin.io/api"
        self.headers = {"Authentication": api_key}

    def create_alias(self, mail_box="default"):
        data = {
            "mailbox": mail_box,
            "hostname": "simplelogin.com"
        }
        response = requests.post(
            f"{self.base_url}/aliases",
            json=data,
            headers=self.headers
        )
        return response.json()
```

## Best Practices for Production Use

When implementing email aliasing in production environments:

- **Use domain catch-all**: Point your domain's catch-all at your alias service to automatically create aliases for any subdomain
- **Implement key rotation**: Regularly rotate PGP keys and update them in your alias service
- **Monitor alias activity**: Check forwarded email logs for suspicious patterns
- **Enable 2FA**: Protect your alias service account with hardware keys or TOTP
- **Separate concerns**: Use different aliases for different purposes (shopping, social media, work)

### Step 5: Comparing Anonaddy vs SimpleLogin

| Feature | Anonaddy | SimpleLogin |
|---------|----------|-------------|
| Free tier aliases | Unlimited (shared domain) | 10 aliases |
| Custom domains | Paid plans | Paid plans |
| Self-hosting | Yes (open source) | Yes (open source) |
| PGP encryption | Built-in | Built-in |
| Browser extension | Yes | Yes |
| Mobile app | Community | Official |
| Reply from alias | Yes | Yes |
| Owned by | Independent | Proton AG |

SimpleLogin's acquisition by Proton AG means it integrates well with ProtonMail. For users who prefer full independence and self-hosting, Anonaddy remains the better option.

### Step 6: Security Hardening for Alias Accounts

Your alias service account is a high-value target. Protect it:

1. **Enable hardware key 2FA** (YubiKey or similar FIDO2 key), not just TOTP
2. **Use a strong, unique password** generated by your password manager
3. **Set up login notifications** to detect unauthorized access
4. **Review active sessions** monthly and revoke any you don't recognize
5. **Enable PGP encryption** so even if the service is compromised, email contents remain encrypted

These precautions ensure that your alias service remains a privacy enhancement rather than a single point of failure.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to set up?**

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

- [Privacy-Focused Email Forwarding Services Comparison](/privacy-tools-guide/privacy-focused-email-forwarding-services-comparison/)
- [How To Check If Your Email Is Being Forwarded](/privacy-tools-guide/how-to-check-if-your-email-is-being-forwarded-without-knowle/)
- [How to Set Up S/MIME Email Encryption: A Practical Guide](/privacy-tools-guide/how-to-set-up-smime-email-encryption/)
- [Email Encryption with GPG](/privacy-tools-guide/gpg-email-encryption-step-by-step)
- [OpenPGP vs S/MIME Email Encryption Comparison](/privacy-tools-guide/openpgp-vs-smime-email-encryption-comparison-which-to-choose/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
