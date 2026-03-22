---
layout: default
title: "Setting Up Encrypted Email with Tutanota"
description: "How to set up Tutanota encrypted email, configure custom domains, use the desktop client, and integrate with external tools for end-to-end encrypted communication"
date: 2026-03-22
author: theluckystrike
permalink: /tutanota-encrypted-email-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Setting Up Encrypted Email with Tutanota

Tutanota (now rebranded as Tuta) encrypts email end-to-end between Tutanota users automatically, and can encrypt to external recipients using a shared password. Unlike ProtonMail, Tutanota also encrypts subject lines — a meaningful privacy detail since subject lines are often more revealing than message bodies.

---

## What Tutanota Encrypts

| Element | Tutanota-to-Tutanota | Tutanota-to-External |
|---------|---------------------|---------------------|
| Email body | E2EE | E2EE (with password) |
| Subject line | E2EE | E2EE (with password) |
| Attachments | E2EE | E2EE (with password) |
| Sender/recipient metadata | Not encrypted | Not encrypted |

"Not encrypted" metadata means the mail servers still see who sent to whom — this is inherent to how email routing works.

---

## Create an Account

Tutanota's free plan includes 1GB storage and one email address. Paid plans start at €3/month and add custom domains, aliases, and more storage.

1. Go to `tuta.com` → Create free account
2. Choose your `@tuta.com` or `@tutanota.com` address
3. Write down your recovery code — this is the only way to recover access if you forget your password. Tutanota cannot reset it for you (they have no access to your encrypted data)
4. Store the recovery code offline, not in a cloud service

---

## Enable Two-Factor Authentication

```
Account settings → Login → Two-factor authentication → Add authenticator app

Scan QR code with:
- Aegis (Android, F-Droid)
- Raivo OTP (iOS)
- KeePassXC TOTP

Save backup codes offline
```

Tutanota also supports hardware security keys (FIDO2/WebAuthn):

```
Account settings → Login → Two-factor authentication → Add security key
→ Insert YubiKey or similar → tap/press the key
```

---

## Desktop Client Setup

```bash
# Linux — download AppImage
wget https://mail.tutanota.com/desktop/tutanota-desktop-linux.AppImage
chmod +x tutanota-desktop-linux.AppImage

# Verify the signature
wget https://mail.tutanota.com/desktop/tutanota-desktop-linux.AppImage.sig
gpg --keyserver keyserver.ubuntu.com --recv-keys 0x7D31293D3B3CAE1B
gpg --verify tutanota-desktop-linux.AppImage.sig tutanota-desktop-linux.AppImage

./tutanota-desktop-linux.AppImage

# Or install via Flatpak
flatpak install flathub com.tutanota.Tutanota
```

**macOS:**

```bash
# Download from tuta.com/download
# Drag to Applications
# First launch: right-click → Open (bypasses Gatekeeper warning)
```

---

## Custom Domain Setup

A custom domain (`you@yourdomain.com`) gives you email independence — if Tutanota were to shut down or you want to move, your address stays yours.

**Requirements**: A domain you control and a paid Tutanota plan.

```
Account settings → Email → Custom email domain → Add domain
```

**DNS records to add at your registrar:**

```bash
# MX records (route mail to Tutanota)
yourdomain.com.  MX  10  mail.tutanota.de.

# SPF record (prevent spoofing)
yourdomain.com.  TXT  "v=spf1 include:spf.tutanota.de -all"

# DKIM record (Tutanota generates the DKIM key — copy from their setup wizard)
# DMARC record
_dmarc.yourdomain.com.  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com"
```

---

## Sending Encrypted Email to External Recipients

When you email someone outside Tutanota, they receive a link to a password-protected message:

```
Compose email → click the lock icon → set an external password
The recipient receives a link + must enter the password to read the message
```

Communicate the password via a separate channel — Signal call, SMS, or pre-arranged password. The recipient can reply encrypted within the same thread.

---

## Tutanota's Encryption Implementation

```
Key generation:
- RSA-4096 + AES-256 for existing accounts
- X25519 (ECC) + AES-256 for new accounts created after 2023

Email encryption flow:
1. Sender's client generates random AES-256 session key
2. Session key encrypts email body, subject, attachments
3. Session key is encrypted with recipient's public key
4. Both encrypted email + encrypted session key sent to server
5. Recipient's client decrypts session key with private key
6. Session key decrypts email
```

---

## Export / Backup Your Emails

```bash
# Export all emails to EML files via Tutanota desktop client
# File → Import/Export → Export → select folder → Export as EML

# Account Settings → Backup → Create backup (exports all data)
# Store the backup in an encrypted location
```

---

## Comparing Tutanota vs ProtonMail

| Feature | Tutanota | ProtonMail |
|---------|----------|-----------|
| Subject line encrypted | Yes | Yes (since 2024) |
| Free storage | 1GB | 500MB |
| IMAP support | No | Yes (via bridge) |
| Custom domain (free) | No | No |
| Onion address | No | Yes |
| Open source | Yes (clients) | Yes (clients) |
| Headquarters | Germany | Switzerland |

---

## Metadata That Tutanota CANNOT Encrypt

Even with E2EE, some metadata remains visible to Tutanota's servers:

```
Unencrypted metadata:
- Your email address and IP address
- Recipient email addresses (required for routing)
- Send/receive timestamps
- Mail size
- Subject line headers in IMAP access

This metadata alone reveals communication patterns. If Tutanota receives a law enforcement request, they can disclose when you emailed someone, but not what you said or any message content.
```

For maximum privacy, pair Tutanota with Tor Browser or a VPN to hide your IP address.

## Using Tutanota with Mobile Devices

The Tutanota mobile app (iOS and Android) provides the same E2EE guarantees as the web and desktop clients:

```bash
# iOS: Download from App Store
# Android: Download from Play Store or F-Droid (open-source build)

# Setup on mobile:
# 1. Install app
# 2. Log in with credentials
# 3. Allow notification permissions for incoming mail alerts
# 4. Configure autodiscovery for push notifications
```

Mobile E2EE continues even for emails sent outside Tutanota—the mobile app handles password-encrypted external emails seamlessly.

## Integration with Standard Email Clients

Tutanota does not support IMAP/POP access because those protocols lack E2EE support. This means:

- Cannot use Gmail client, Thunderbird, or Outlook with Tutanota
- Must use Tutanota's official clients (web, desktop, mobile)
- Exception: Tutanota offers a mail bridge (beta) for limited IMAP support on desktop

For users requiring Outlook or Thunderbird integration, ProtonMail's mail bridge provides this at the cost of additional complexity.

## Tutanota Business/Organization Plans

Organizations with custom domain requirements and multiple accounts use Tutanota Business:

```bash
# Business plan includes:
- Unlimited aliases per account
- Delegation (managing other mailboxes)
- Group calendars
- Multi-user organization management
- Admin audit logs

# API for programmatic access
curl -X GET https://mail.tutanota.com/api/users \
  -H "Authorization: Bearer $TUTANOTA_API_TOKEN"
```

Business plans start at €5/user/month and suitable for teams prioritizing encrypted collaboration.

## Testing Tutanota's Encryption

For developers who want to verify encryption claims:

```bash
# Send yourself a test email and inspect the network traffic
# Using browser DevTools (F12):
# 1. Open Network tab
# 2. Compose an email
# 3. Watch POST request to /api/mail/send
# 4. Inspect the payload — it's base64 encoded binary encrypted data
# 5. No plaintext email body visible in network traffic

# Attempt to intercept between client and server:
mitmproxy --mode regular --ssl-insecure
# Browser proxy configured to mitmproxy:8080
# Send an email through Tutanota
# mitmproxy shows encrypted request body — no plaintext
```

This verification demonstrates that encryption happens on your device before data leaves your computer.

## Comparison Table: Encrypted Email Providers 2026

| Provider | E2EE | Subject Encrypted | Open Source | Custom Domain (Free) | Jurisdiction |
|----------|------|-----------------|------------|---------------------|--------------|
| Tutanota | Yes | Yes | Yes (clients) | No | Germany |
| ProtonMail | Yes | Yes (since 2024) | Yes (clients) | No | Switzerland |
| Mailfence | Yes | Yes | Partial | No | Belgium |
| Posteo | No | No | No | Yes | Germany |
| StartMail | Yes | No | No | No | Netherlands |

## Threat Model Considerations

Before choosing Tutanota, assess your threat model:

**Tutanota is ideal if:**
- You want subject line encryption
- You value open-source verification
- Your contacts already use encrypted email
- You're in an EU jurisdiction (data residency benefits)
- You don't need IMAP integration

**Consider alternatives if:**
- You need wider ecosystem compatibility (ProtonMail has better Android integration)
- You require legacy email client support (Posteo + Tor)
- You need quantum-resistant encryption (not yet available in any mainstream provider)

## Automation and Integration Scripts

For developers managing multiple Tutanota accounts:

```bash
#!/bin/bash
# Batch export emails from Tutanota accounts
# Note: Tutanota API doesn't support bulk export; use desktop client

# Desktop client export workflow (requires GUI)
# This is a limitation of Tutanota's current architecture
# API access is beta/limited

# For now, scripted exports use:
# 1. Selenium + browser automation
# 2. Tutanota desktop client with file system integration
# 3. Manual export for critical emails
```

Tutanota's lack of full API prevents advanced automation that ProtonMail offers. This is a tradeoff for the simpler security model.

## Frequently Asked Questions

**Can Tutanota read my emails?**

No. Tutanota operates on a zero-knowledge architecture. They cannot read your encrypted emails, access your encryption keys, or decrypt content even with a valid court order.

**What if I forget my password?**

You cannot reset it. Tutanota has no password recovery because they don't store password-recovery mechanisms. This is the intended security design. Always save your recovery code offline. Losing both your password and recovery code means permanent loss of vault access.

**How are calendar events encrypted?**

Tutanota encrypts calendar events end-to-end the same as emails. Shared calendars use key sharing similar to password-protected external emails.

**Can I migrate from ProtonMail to Tutanota?**

Yes, but not automatically. Export emails from ProtonMail as EML files, then import them into Tutanota desktop client. This is a manual process without built-in migration tools.

## Related Reading

- [ProtonMail vs Tutanota for Daily Email Use](/privacy-tools-guide/protonmail-vs-tutanota-for-daily-email-use-honest-comparison/)
- [Anonymous Email Over Tor Setup Guide](/privacy-tools-guide/anonymous-email-over-tor-setup-guide/)
- [Encrypted Messaging for Journalists Guide](/privacy-tools-guide/encrypted-messaging-for-journalists-guide/)
- [PGP Email Encryption Setup Guide](/privacy-tools-guide/pgp-email-encryption-setup/)
- [Email Encryption Standards SMTP TLS vs End-to-End](/privacy-tools-guide/email-encryption-standards/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
