---
layout: default
title: "Setting Up Encrypted Email with Tutanota"
description: "How to set up Tutanota encrypted email, configure custom domains, use the desktop client, and integrate with external tools for end-to-end encrypted communication"
date: 2026-03-22
author: theluckystrike
permalink: /tutanota-encrypted-email-setup-guide/
categories: [guides, privacy]
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

## Related Reading

- [ProtonMail vs Tutanota for Daily Email Use](/privacy-tools-guide/protonmail-vs-tutanota-for-daily-email-use-honest-comparison/)
- [Anonymous Email Over Tor Setup Guide](/privacy-tools-guide/anonymous-email-over-tor-setup-guide/)
- [Encrypted Messaging for Journalists Guide](/privacy-tools-guide/encrypted-messaging-for-journalists-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
