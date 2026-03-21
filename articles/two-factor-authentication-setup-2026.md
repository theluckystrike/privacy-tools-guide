---
layout: default
title: "Two-Factor Authentication Setup Guide 2026"
description: "Set up 2FA properly in 2026. Covers TOTP apps, hardware keys, passkeys, and backup codes. Includes setup for Google, GitHub, and SSH."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: two-factor-authentication-setup-2026
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A strong password is no longer enough. Credential stuffing attacks, phishing campaigns, and data breaches mean that passwords leak constantly. Two-factor authentication (2FA) means an attacker who gets your password still can't get in without your second factor.

This guide covers every common 2FA method, how to set up each one, and how to avoid the traps that leave people locked out of their own accounts.

## 2FA Methods Ranked by Security

From strongest to weakest:

1. **Hardware security keys (FIDO2/WebAuthn)** — YubiKey, Google Titan, etc. Phishing-resistant. Best option.
2. **Passkeys** — Device-bound FIDO2 credentials stored in a secure enclave. Phishing-resistant.
3. **TOTP apps** — 6-digit codes from apps like Aegis, Raivo, or Google Authenticator. Good protection.
4. **Push notifications** — Approve/deny prompts from Duo or Microsoft Authenticator. Vulnerable to push fatigue attacks.
5. **SMS codes** — Vulnerable to SIM swapping, SS7 attacks, and phishing. Avoid for sensitive accounts.

If a service only offers SMS 2FA, it's better than nothing — but push for TOTP or hardware key support.

## Method 1: TOTP Apps

TOTP (Time-based One-Time Password) apps generate a fresh 6-digit code every 30 seconds using a shared secret.

### Recommended Apps

- **Aegis Authenticator** (Android) — open source, encrypted backup, import/export
- **Raivo** (iOS) — open source, iCloud sync, encrypted export
- **2FAS** (iOS/Android) — cross-platform, open source
- Avoid Google Authenticator and Authy — both have privacy or lock-in concerns

### Setting Up TOTP on a New Account

1. Go to your account's security settings
2. Find "Two-Factor Authentication" or "Authenticator App"
3. Scan the QR code with your TOTP app
4. Enter the 6-digit code to verify
5. Save the backup/recovery codes (explained below)

### Setting Up TOTP on GitHub

```bash
# Verify 2FA is active after setup
gh auth status
# Should show 2FA is required for this account
```

Via web:
1. GitHub > Settings > Password and Authentication > Enable two-factor authentication
2. Choose "Authenticator app"
3. Scan the QR code
4. Confirm with a code from the app
5. Download backup codes

### Backup Your TOTP Seeds

The QR code is a base32-encoded secret seed. If your phone is lost or broken, you need this seed to recover access.

**Export from Aegis:**
- Aegis > Settings > Backups > Export > Plain text export (decrypt and store in password manager)

**Export from Raivo:**
- Raivo > Settings > Export OTP tokens to ZIP

Store the export file encrypted in your password manager or an encrypted volume. Never in plaintext cloud storage.

To manually decode a TOTP QR code:

```bash
# Install qr-decode tool
sudo apt install zbar-tools   # Ubuntu
brew install zbar             # macOS

# Decode a QR image
zbarimg --raw totp-qr.png
# Outputs: otpauth://totp/Example:user@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Example
```

The `secret=` value is what you need to store.

## Method 2: Hardware Security Keys

FIDO2/WebAuthn keys are the strongest form of 2FA. They're cryptographic devices — even if a phishing site captures your password, it can't use the hardware key response because the key only responds to the legitimate domain.

### Recommended Keys

- **YubiKey 5 Series** — widest protocol support (FIDO2, PIV, OpenPGP, TOTP, OTP)
- **Google Titan Key** — FIDO2 only, simpler, cheaper
- **Nitrokey 3** — open source hardware

**Buy at least two keys** — one primary, one backup. Register both everywhere. Store the backup somewhere safe.

### Register a YubiKey on GitHub

1. Settings > Password and Authentication > Security Keys > Add a security key
2. Name the key (e.g., "YubiKey 5C primary")
3. When prompted, insert the YubiKey and tap it
4. Registration completes

### Use YubiKey for SSH Authentication

The YubiKey can hold FIDO2 SSH keys in its secure enclave:

```bash
# Generate an SSH key stored on the YubiKey (requires OpenSSH 8.2+)
ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_yubikey -C "yubikey-ssh"

# This creates a key stub on disk — the private key never leaves the YubiKey
# Upload public key to server
ssh-copy-id -i ~/.ssh/id_ed25519_yubikey.pub user@server

# Connect (will prompt for YubiKey tap)
ssh -i ~/.ssh/id_ed25519_yubikey user@server
```

### TOTP on YubiKey

YubiKeys can store TOTP secrets with the `ykman` tool:

```bash
# Install ykman
sudo apt install yubikey-manager   # Ubuntu
brew install ykman                 # macOS

# List existing TOTP accounts on the key
ykman oath accounts list

# Add a TOTP account
ykman oath accounts add "GitHub:user@example.com" JBSWY3DPEHPK3PXP

# Get a code
ykman oath accounts code "GitHub:user@example.com"
```

This stores TOTP seeds on the hardware key rather than your phone — they can't be phished or exported without physical access.

## Method 3: Passkeys

Passkeys are FIDO2 credentials stored in your device's secure enclave (Secure Enclave on Apple, TPM on Windows, OS keystore on Android). They replace passwords entirely for supported sites.

### Set Up a Passkey on a Supported Site

Sites supporting passkeys in 2026: GitHub, Google, Apple, Microsoft, PayPal, Amazon, Coinbase, and many others.

On GitHub:
1. Settings > Password and Authentication > Passkeys > Add a passkey
2. Follow device prompts (Face ID, Touch ID, Windows Hello, or hardware key)
3. The passkey is created and linked to your account

Passkeys sync across devices via:
- iCloud Keychain (Apple devices)
- Google Password Manager (Android/Chrome)
- 1Password, Bitwarden (cross-platform)

To use passkeys in a password manager:

```bash
# Bitwarden CLI — list passkeys
bw list items --search "passkey" | jq '.[] | select(.type == 7)'

# 1Password supports passkey creation and storage via browser extension
```

## Backup Codes: The Safety Net

Every service that offers 2FA also provides backup codes — single-use codes for account recovery if you lose your 2FA device.

**How to handle backup codes:**

1. Download or print them immediately after enabling 2FA
2. Store in your password manager as a secure note attached to the account entry
3. Print one copy and store in a secure physical location
4. Never photograph them with your phone
5. Regenerate them after using one

Backup codes are not a shortcut — they're a recovery mechanism for genuine emergencies. If you're reaching for backup codes regularly, something is wrong with your 2FA setup.

## Enable 2FA on Critical Accounts

Priority order:

1. **Email** (Google/Microsoft/ProtonMail) — controls everything else
2. **Password manager** (Bitwarden, 1Password) — protects all other passwords
3. **Code repositories** (GitHub, GitLab) — contains your work and sometimes secrets
4. **Financial accounts** (bank, brokerage, PayPal) — obvious
5. **Domain registrar** (Cloudflare, Namecheap) — DNS hijacking is catastrophic
6. **Cloud providers** (AWS, GCP, Azure) — infrastructure access
7. **Social media** — impersonation and recovery phone abuse

Check which accounts have 2FA enabled (and which don't) at `https://2fa.directory`.

## Common Mistakes

**Using SMS as a fallback** — If you enable TOTP but keep SMS as a fallback, attackers can SIM swap and bypass your TOTP. Remove SMS fallback after enabling a stronger method.

**Storing TOTP app backup in the same account** — Don't back up your email TOTP seed only to your email account's cloud backup. Circular dependency.

**Only registering one hardware key** — If you lose it, you're locked out. Always register two.

**Not storing backup codes** — Skipping this step is the single most common reason people get locked out of accounts permanently.


## Related Articles

- [Dating App Two Factor Authentication Setup Protecting Accoun](/privacy-tools-guide/dating-app-two-factor-authentication-setup-protecting-accoun/)
- [ProtonMail Two-Factor Authentication Guide](/privacy-tools-guide/protonmail-two-factor-authentication-guide/)
- [How To Set Up Offline Encrypted Communication Between Two Pe](/privacy-tools-guide/how-to-set-up-offline-encrypted-communication-between-two-pe/)
- [How To Set Up Vpn Failover Between Two Providers Automatical](/privacy-tools-guide/how-to-set-up-vpn-failover-between-two-providers-automatical/)
- [Dkim Spf Dmarc Email Authentication How They Protect Against](/privacy-tools-guide/dkim-spf-dmarc-email-authentication-how-they-protect-against/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
