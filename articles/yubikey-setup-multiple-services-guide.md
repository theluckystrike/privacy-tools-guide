---
---
layout: default
title: "YubiKey Setup for Multiple Services Guide"
description: "How to configure a single YubiKey for SSH authentication, GPG signing, TOTP codes, passkeys, and password manager 2FA — step-by-step for YubiKey 5 series"
date: 2026-03-21
author: theluckystrike
permalink: /yubikey-setup-multiple-services-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A YubiKey can serve multiple authentication roles simultaneously — SSH key, GPG signing key, TOTP generator, FIDO2/passkey authenticator, and password manager second factor. Most people configure it for only one or two of these. This guide covers setting up all major use cases on a single YubiKey 5 series device.

## Prerequisites

```bash
# Install YubiKey management tools
# Linux (Debian/Ubuntu)
sudo apt install yubikey-manager yubioath-desktop libpam-u2f

# macOS
brew install ykman

# Verify YubiKey is detected
ykman info
```

The output should show your YubiKey model, serial number, and firmware version.

### Step 1: Understand YubiKey Interfaces

The YubiKey 5 supports multiple independent interfaces on the same device:

- **OTP** (slot 1 and 2) — One-touch OTP and static passwords
- **FIDO2** — Passkeys and hardware-bound 2FA (the modern standard)
- **PIV** — Smart card standard for certificates, SSH keys
- **OpenPGP** — GPG key storage
- **OATH** — TOTP/HOTP codes (compatible with Google Authenticator, Authy)

These interfaces do not interfere with each other. You can use all of them on one key.

### Step 2: Part 1: SSH Authentication with PIV

Using YubiKey's PIV interface for SSH gives you hardware-backed SSH keys that cannot be extracted from the device.

### Generate a PIV key on the YubiKey

```bash
# Change the default PIV PIN (default: 123456) and PUK (default: 12345678)
ykman piv access change-pin
ykman piv access change-puk

# Set a management key (used for key operations, not daily use)
ykman piv access change-management-key --generate --protect

# Generate a 2048-bit RSA key in slot 9a (authentication slot)
ykman piv keys generate --algorithm RSA2048 9a public_key.pem

# Create a self-signed certificate for the key
ykman piv certificates generate --subject "CN=SSH Key" 9a public_key.pem
```

### Configure SSH to use the PIV key

```bash
# Install pkcs11 provider for SSH
sudo apt install libykcs11

# Export the public key in OpenSSH format
ssh-keygen -D /usr/lib/x86_64-linux-gnu/libykcs11.so -e

# Add this public key to authorized_keys on target servers
# Then configure your SSH client to use the PKCS11 provider:
```

Add to `~/.ssh/config`:
```
Host *
    PKCS11Provider /usr/lib/x86_64-linux-gnu/libykcs11.so
```

When you SSH to a server, you'll be prompted to touch the YubiKey. The private key never leaves the hardware.

### Step 3: Part 2: GPG Signing with OpenPGP Interface

YubiKey stores GPG subkeys in its OpenPGP applet. The master key remains off-device; only signing, encryption, and authentication subkeys live on the YubiKey.

### Set up GPG with YubiKey

```bash
# First, create a master key (do this on an air-gapped machine ideally)
gpg --expert --full-generate-key
# Choose: (8) RSA (set your own capabilities)
# For master key: enable only Certify
# Key size: 4096
# Expiry: 2y

# Create subkeys for Sign, Encrypt, Authenticate
# In gpg --edit-key: addkey → (8) RSA → select appropriate capability
# Repeat for each subkey type

# Export the master key backup BEFORE moving keys to YubiKey
gpg --armor --export-secret-keys KEYID > master-backup.asc
# Store this securely offline

# Move subkeys to YubiKey
gpg --edit-key KEYID
gpg> key 1     # Select sign subkey
gpg> keytocard # Move to signing slot
gpg> key 2     # Select encrypt subkey
gpg> keytocard # Move to encryption slot
gpg> key 3     # Select auth subkey
gpg> keytocard # Move to auth slot
gpg> save
```

After this, the YubiKey contains the subkeys and GPG will use them automatically for signing and encryption operations. Touching the key is required for each operation.

```bash
# Verify keys are on card
gpg --card-status

# Test signing
echo "test" | gpg --sign
# Should prompt for PIN and require physical touch
```

### Step 4: Part 3: TOTP Codes with OATH Interface

The YubiKey can generate TOTP codes (the 6-digit codes used for 2FA) using its OATH applet. This is different from push-based authentication — the codes are generated on the key itself.

```bash
# Add a TOTP account
ykman oath accounts add --touch "GitHub" SECRETBASE32KEY

# The --touch flag requires a physical touch to show the code (recommended)

# List stored accounts
ykman oath accounts list

# Generate a code
ykman oath accounts code GitHub
```

### Using the Yubico Authenticator GUI

The Yubico Authenticator app (available on Windows, macOS, Linux, iOS, Android) provides a graphical interface for OATH codes and requires the YubiKey to be present to display them. This is significantly more secure than Authy or Google Authenticator because the codes are generated on the hardware and disappear when the key is removed.

```bash
# Linux install
sudo apt install yubioath-desktop
```

### Step 5: Part 4: FIDO2 and Passkeys

FIDO2 is the most modern authentication standard. The YubiKey stores resident credentials (passkeys) in its FIDO2 applet.

```bash
# Check FIDO2 status
ykman fido info

# Set FIDO2 PIN (required for passkeys)
ykman fido access change-pin

# List stored passkeys
ykman fido credentials list

# Reset FIDO2 applet if needed (warning: deletes all credentials)
ykman fido reset
```

For websites, FIDO2/passkey registration happens through the browser when you add a security key in an account's security settings. No command-line setup needed — just plug in the YubiKey when prompted.

### Step 6: Part 5: Password Manager Second Factor

For most password managers (Bitwarden, 1Password, etc.), YubiKey functions as a FIDO2 security key. Register it in the security settings of your password manager account.

For Bitwarden specifically:

1. Go to bitwarden.com → Account Settings → Security → Two-step Login
2. Click "Manage" next to "FIDO2 WebAuthn"
3. Insert YubiKey and click "Read Key"
4. Give the key a name and save

You can register two or more YubiKeys as backups. This is important — if you lose your only key, you could be locked out.

### Step 7: Manage Multiple Keys

If you have a backup YubiKey, configure it with the same credentials where possible:

- **SSH/PIV**: Generate a separate key pair on the backup key; add both public keys to `authorized_keys`
- **GPG**: Store the same subkeys on both keys (if key was moved with `keytocard`, use backup subkey copy)
- **TOTP**: Add the same TOTP secrets to both keys during initial setup (before they expire)
- **FIDO2/Passkeys**: Register both keys separately with each service

```bash
# Script to check which YubiKey is currently connected
ykman info | grep "Serial number"

# Label your keys physically and track which serial is which
```

### Step 8: Revoking a Lost YubiKey

If a YubiKey is lost:

```bash
# For FIDO2 credentials: remove the key from each service's security settings
# (no central revocation — must be done per-service)

# For GPG: if the subkey had an expiry date, it will expire automatically
# If no expiry, revoke the subkey:
gpg --edit-key KEYID
gpg> key 1
gpg> revkey
gpg> save
gpg --send-keys KEYID  # Publish revocation to keyserver

# For SSH: remove the lost key's public key from all authorized_keys files
```

This is why having two YubiKeys registered for every service matters — losing one key should not lock you out.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Articles

- [How to Use YubiKey for Maximum Security in 2026](/privacy-tools-guide/how-to-use-yubikey-for-maximum-security-2026/)
- [How to Use YubiKey for SSH Authentication](/privacy-tools-guide/articles/how-to-use-yubikey-for-ssh-authentication-guide/)
- [How to Use Password Manager with YubiKey Hardware Key Setup](/privacy-tools-guide/how-to-use-password-manager-with-yubikey-hardware-key-setup/)
- [YubiKey vs Titan Security Key: A Developer Comparison](/privacy-tools-guide/yubikey-vs-titan-security-key-comparison/)
- [How to Harden SSH Server Configuration](/privacy-tools-guide/how-to-harden-ssh-server-configuration/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to multiple services guide?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
