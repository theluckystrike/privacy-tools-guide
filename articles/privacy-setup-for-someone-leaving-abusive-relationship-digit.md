---
layout: default
title: "Privacy Setup for Someone Leaving Abusive Relationship: Digital Safety Guide"
description: "A practical technical guide for securing digital privacy when leaving an abusive relationship. Covers device hardening, account isolation, and communication security for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /privacy-setup-for-someone-leaving-abusive-relationship-digit/
---

{% raw %}

Leaving an abusive relationship introduces specific digital security challenges that differ from typical privacy hardening. Abusers often have established access to devices, accounts, and can leverage social engineering against shared services. This guide addresses the unique threat model faced when transitioning to safety.

## Assessing Access Vectors

An abuser may have obtained access through multiple pathways. Before securing anything, systematically identify what needs to be revoked:

**Physical access** includes any device the abuser could have touched—phones, laptops, tablets, or smart home equipment. They may have installed monitoring applications or configured recovery options.

**Account access** spans email, social media, cloud storage, and financial accounts. Check for:
- Authorized devices listed in account security settings
- Recovery email addresses or phone numbers you don't recognize
- Third-party app permissions granting access to data
- Shared passwords between accounts

**Social engineering vectors** involve the abuser leveraging mutual contacts, impersonating you to service providers, or using personal information to bypass security questions.

## Immediate Device Hardening

### Factory Reset Protocol

When possible, factory reset all devices that the abuser had physical access to. This removes any hidden monitoring software. For mobile devices:

```bash
# On Android, ensure encryption is enabled before reset
# Settings > Security > Encryption

# On iOS, sign out of iCloud before erasing
# Settings > [Your Name] > Sign Out
```

For laptops, boot into recovery mode and wipe the storage drive. Reinstall the operating system from trusted media rather than relying on recovery partitions that could harbor firmware-level compromises.

### Mobile Device Configuration

Configure a new device with a fresh account rather than restoring from backups that may contain compromises. Enable these settings immediately:

- **Two-factor authentication** on all critical accounts—use an authenticator app rather than SMS
- **Biometric authentication** for device unlock, with a strong PIN as backup
- **Encrypted messaging** for all communications—Signal provides default end-to-end encryption
- **Location services** disabled by default, enabled only when explicitly needed

Create a new Apple ID or Google account on fresh devices. Do not use your previous credentials.

## Account Isolation Strategy

### Email Account Segregation

Create entirely new email accounts for all sensitive communications. Use a privacy-respecting provider with strong encryption:

```bash
# Example: Generate a GPG keypair for encrypted email communication
gpg --full-generate-key
# Select RSA 4096-bit, no expiration for simplicity
# Use a strong passphrase stored in your password manager
```

Configure your new email with the following security measures:
- App-specific passwords disabled
- IMAP/POP access reviewed and removed if unnecessary
- Recovery options set to your new phone number and email only
- Login notifications enabled

### Password Manager Architecture

Your password manager becomes critical infrastructure. Generate completely new passwords for every account:

```bash
# Example: Using bitwarden CLI to generate secure passwords
bw generate --length 24 --uppercase --lowercase --number --symbol
```

Organize vault entries with clear naming conventions. Consider creating separate vaults:
- **Financial** — banking, investment, credit card portals
- **Communication** — email, messaging, video calling
- **Identity** — government services, healthcare portals
- **Recovery** — password reset accounts

Enable emergency access with a trusted contact who understands the sensitivity of this arrangement.

### Two-Factor Authentication Migration

Review every account with 2FA enabled. For each:
1. Verify the registered devices are only those you control
2. Regenerate TOTP secrets where compromise is possible
3. Remove phone numbers used as SMS backup if they could be SIM-swapped
4. Store backup codes in a secure physical location

Hardware security keys provide the strongest 2FA option. If using YubiKey or similar:

```bash
# Verify FIDO2 registration on supported services
# Check account security settings for "Security Key" or "Hardware Token" options
```

## Communication Security

### Encrypted Messaging Setup

Signal serves as the primary recommendation for secure communication. Configure these settings:

- Enable **disappearing messages** with a short duration (1 hour or 24 hours)
- Set **registration lock** to require your PIN if SIM is transferred
- Disable **link previews** to prevent metadata leakage
- Use **Signal PIN** to protect local database

For sensitive communications, consider:

```python
# Example: Using Python to encrypt messages with recipient's public key
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.backends import default_backend

def encrypt_message(message, recipient_public_key):
    ciphertext = recipient_public_key.encrypt(
        message.encode(),
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )
    return ciphertext
```

### Email Encryption

For sensitive written communication, configure GPG encryption:

```bash
# Export your public key to share with trusted contacts
gpg --armor --export your@email.com > public_key.asc

# Encrypt a message for a recipient
gpg --encrypt --armor --recipient recipient@email.com --output message.asc message.txt
```

Provide your public key to trusted contacts through an in-person exchange or secure channel.

## Financial Account Protection

### Bank Account Isolation

Contact your bank to:
- Add a verbal password or PIN for in-person and phone transactions
- Remove any authorized users you did not add
- Enable transaction alerts via your new email and phone
- Review and close any recurring payments to shared services

Consider opening a new account at a different institution if your current bank has security gaps.

### Credit Freeze

Place a credit freeze with all major bureaus:

```bash
# Equifax: https://www.equifax.com/personal/credit-report-services/
# Experian: https://www.experian.com/freeze/center.html
# TransUnion: https://www.transunion.com/credit-freeze
```

This prevents an abuser from opening accounts in your name using stolen personal information.

## Location Privacy

### Device Location Settings

Disable location sharing across all platforms:

- **Google Maps**: Turn off Location History and Timeline
- **Apple**: Disable Significant Locations in Privacy settings
- **Social media**: Review and disable geotagging on posts
- **Photos**: Strip EXIF metadata before sharing images

### Smart Home Isolation

If the abuser had access to smart home devices, treat them as compromised:

```bash
# Factory reset all smart home devices
# Change WiFi credentials
# Create a new network for smart devices
# Use VLAN segmentation if your router supports it
```

Replace any device that cannot be factory reset or whose firmware cannot be verified.

## Social Media Hygiene

### Account Audit

Review every social media account:
- Remove the abuser and their associates
- Check tagged photos and posts
- Review login history for unauthorized access
- Remove third-party app permissions
- Enable login alerts

Set accounts to private. Consider a complete break from social media during the transition period.

### Search Engine Removal

Request removal of personal information from search results:

```bash
# Google removal request form
# https://www.google.com/webmasters/tools/removals
```

This prevents an abuser from finding new accounts or location information through search.

## Documentation and Monitoring

### Security Checklist

Maintain a written record of:
- All changed passwords (stored in password manager)
- New account creations
- Devices reset or replaced
- Contacts with financial institutions
- Police reports or restraining orders

### Ongoing Monitoring

Set up automated monitoring:
- HaveIBeenPwned.com alerts for your new email
- Google Alerts for your name and new phone number
- Credit report monitoring (many services offer free monitoring)

## Summary

Securing digital life after leaving an abusive relationship requires systematic action across devices, accounts, and communication channels. The priority order:

1. Create new devices and accounts, isolated from previous infrastructure
2. Enable strong authentication everywhere—hardware keys preferred
3. Encrypt all communications using Signal and GPG
4. Freeze credit and secure financial accounts
5. Audit social media and remove location sharing

These measures provide defense-in-depth against an abuser who may attempt continued digital surveillance or identity theft.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
