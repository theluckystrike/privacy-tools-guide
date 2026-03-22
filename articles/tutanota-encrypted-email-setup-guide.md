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

"Not encrypted" metadata means the mail servers still see who sent to whom — this is inherent to how email routing works. No email provider, including Tutanota, can hide the fact that a message traveled between two addresses.

---

## Create an Account

Tutanota's free plan includes 1GB storage and one email address. Paid plans start at €3/month and add custom domains, aliases, and more storage. The Revolutionary plan at €6/month includes 20 custom domains and up to 100 aliases, which is useful if you want compartmentalized identities for different purposes.

1. Go to `tuta.com` → Create free account
2. Choose your `@tuta.com` or `@tutanota.com` address
3. Write down your recovery code — this is the only way to recover access if you forget your password. Tutanota cannot reset it for you (they have no access to your encrypted data)
4. Store the recovery code offline, not in a cloud service — a printed sheet kept somewhere secure is the right approach
5. On mobile: install the official Tuta app from F-Droid, Google Play, or App Store. Avoid third-party clients that claim IMAP support — Tutanota does not expose IMAP, so anything claiming to offer it is using a workaround that compromises security

---

## Enable Two-Factor Authentication

Adding a second factor dramatically reduces the risk from password leaks. Tutanota supports both authenticator apps (TOTP) and hardware security keys.

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

If you add a hardware key, also add an authenticator app as a fallback. Losing your only 2FA method locks you out permanently — Tutanota cannot bypass this for you.

---

## Desktop Client Setup

The Tutanota desktop client is built on Electron and signed by Tutanota GmbH. Verify the signature before running any downloaded binary.

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

**Windows:**

Download the installer from `tuta.com/download`. Tutanota publishes SHA256 checksums for each release on their GitHub Releases page (`github.com/tutao/tutanota/releases`). Run `certutil -hashfile TutanotaInstaller.exe SHA256` and compare before installing.

The desktop client stores a local cache of your emails in an encrypted SQLite database. This means you can search and read cached mail offline. The cache is protected by your login password — do not store your Tutanota password in a browser password manager or allow anyone else to access your user account on the same machine.

---

## Custom Domain Setup

A custom domain (`you@yourdomain.com`) gives you email independence — if Tutanota were to shut down or you want to move, your address stays yours.

**Requirements**: A domain you control and a paid Tutanota plan (Legends tier or above).

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

After saving DNS records, propagation can take up to 48 hours, though it usually completes within an hour. Use `dig MX yourdomain.com` to verify the MX record is live before completing setup in Tutanota's wizard.

Once the domain is verified, you can create aliases at that domain directly in the account settings. This lets you use `support@yourdomain.com`, `personal@yourdomain.com`, and similar addresses, all received in the same inbox. Aliases can be hidden from your contact list to avoid exposing the main address.

---

## Sending Encrypted Email to External Recipients

When you email someone outside Tutanota, they receive a link to a password-protected message hosted on Tutanota's servers:

```
Compose email → click the lock icon → set an external password
The recipient receives a link + must enter the password to read the message
```

Communicate the password via a separate channel — Signal call, SMS, or pre-arranged password. The recipient can reply encrypted within the same thread, and their reply is also E2EE. If you communicate regularly with the same external contact, you can save the password per-contact in Tutanota's settings so you don't have to set it each time.

A common pattern for journalists or lawyers is to publish a Tutanota address with a pre-arranged external encryption password on a contact page, so sources can initiate encrypted contact without needing a Tutanota account.

---

## Tutanota's Encryption Implementation

Tutanota's encryption is audited and open source. The client code is available at `github.com/tutao/tutanota`.

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

Tutanota does not support PGP — this is by design. PGP leaves subject lines unencrypted and exposes public key metadata. Tutanota's native encryption provides better metadata protection. The trade-off is interoperability: you cannot receive PGP-encrypted mail from Thunderbird/GPG users directly. If PGP interoperability is a requirement, ProtonMail (via its Bridge) is a better fit.

---

## Export / Backup Your Emails

Tutanota's desktop client includes a basic export function. For a more systematic backup, use the Tutanota export functionality periodically:

```bash
# Export all emails to EML files via Tutanota desktop client
# File → Import/Export → Export → select folder → Export as EML

# Account Settings → Backup → Create backup (exports all data)
# Store the backup in an encrypted location
```

EML files are plain MIME-formatted text and are importable into Thunderbird, Apple Mail, and most mail clients. Store backups in an encrypted volume — on macOS, use a disk image created with Disk Utility (Format: Mac OS Extended, Encryption: AES-256). On Linux, use a LUKS-encrypted container or Cryptomator vault.

If you have years of email and large attachments, export by folder and compress each folder before archiving. Tutanota's export does not currently resume interrupted exports, so running it on a fast, stable connection matters for large mailboxes.

---

## Aliases and Spam Compartmentalization

One practical use for Tutanota is creating aliases for different contexts — shopping, newsletters, professional contacts — while all mail lands in one inbox. This way, if one alias gets spammed, you delete it and create a new one without losing your main address.

```
Account settings → Email → Email aliases → Add alias
(Paid plan required for more than one additional alias)
```

For free-plan users, Tutanota offers `+` subaddressing at some address domains: `yourname+alias@tuta.com`. Not all senders honor the full address for filtering, so paid aliases are more reliable.

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
| PGP support | No | Yes |
| Calendar (E2EE) | Yes | Yes |

Germany's privacy laws (GDPR, BDSG) provide strong legal protections, though Germany participates in 14 Eyes intelligence sharing. Switzerland's laws are slightly stronger for non-EU data retention requests. Neither affects the practical security of your email if the encryption is implemented correctly — an attacker who compromises Tutanota's servers gets ciphertext they cannot read.

---

## Related Reading

- [ProtonMail vs Tutanota for Daily Email Use](/privacy-tools-guide/protonmail-vs-tutanota-for-daily-email-use-honest-comparison/)
- [Anonymous Email Over Tor Setup Guide](/privacy-tools-guide/anonymous-email-over-tor-setup-guide/)
- [Encrypted Messaging for Journalists Guide](/privacy-tools-guide/encrypted-messaging-for-journalists-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
