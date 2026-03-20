---
layout: default
title: "Privacy Setup For Someone Leaving Abusive Relationship Digit"
description: "A technical guide for developers and power users helping someone leave an abusive relationship. Covers device hardening, account security, secure."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /privacy-setup-for-someone-leaving-abusive-relationship-digit/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Help abuse survivors immediately by enabling full-disk encryption on their device, securing their email and primary accounts with new strong passwords and two-factor authentication, removing any location-sharing features and checking for installed tracking apps, and establishing secure communications channels using Signal or Tor. Change account recovery methods, remove the abuser from all shared accounts, and document threatening communications for legal purposes. This guide provides practical implementation patterns for securing devices, communications, and accounts during the critical transition period, with actionable steps for technically-minded supporters.

## Immediate Threat Assessment

Before implementing any security measures, understand the common attack vectors in abusive situations:

- **Shared device access**: Abusers may have physical access to victim devices or know passwords
- **Account recovery exploitation**: Email account recovery can grant access to other services
- **Location tracking**: Find My features, GPS data in photos, or installed tracking apps
- **Social engineering**: Abusers may contact support teams pretending to be the victim
- **SIM swapping**: Phone number porting to seize control of authentication

The goal is creating immediate separation while building longer-term security posture. Start with the highest-impact changes first.

## Device Hardening

### Removing Tracker Access

First, audit all location-sharing settings across devices. On iOS, check Find My settings comprehensively:

```bash
# Check shared location settings via Apple ID
# 1. Settings > Apple ID > Find My
# 2. Remove any sharing with the abuser's Apple ID
# 3. Disable Share My Location entirely if not needed
```

On Android, similar checks apply for Google Location Sharing and any third-party tracking apps. Look for:
- Apps with accessibility permissions (often used for spying)
- Unknown device management profiles
- Location history enabled for timeline features

### Factory Reset as Baseline

When device compromise is suspected and immediate safety is priority, factory reset provides a clean slate. However, this introduces risks—abusers may notice and escalate. Consider:

```bash
# Before reset, backup ONLY essential data:
# - Contact numbers (manually recreate, don't restore full backup)
# - Photos to encrypted USB (then securely wipe device)
# - Any documents to encrypted storage
#
# DO NOT restore from iCloud/Google backup - these may contain
# tracking data or间谍 software
```

For Android, verify the bootloader is locked after reset. Enter recovery mode and confirm:
- Bootloader status shows "locked" 
- Verified boot shows "green" or "verified"

### New Device Procurement

If possible, obtain a fresh device using cash or a separate payment method. When this isn't feasible, thoroughly reset existing devices. The key principle: assume compromise until proven otherwise.

## Account Security Implementation

### Email Account Isolation

Email is the linchpin of account security. Password resets for almost every service flow through email. Create a new email account with these specifications:

```bash
# New email account setup checklist:
# 1. Use a privacy-respecting provider (ProtonMail, Tutanota)
# 2. Register with phone number from new SIM (see below)
# 3. Enable two-factor authentication with authenticator app
# 4. Set up recovery email to NEW account only
# 5. Disable email forwarding from old account
# 6. Review and remove any connected devices/sessions
```

The new email should be used exclusively for security-critical communications. Do not link it to social accounts the abuser knows about.

### Password Manager Deployment

A password manager enables unique, strong passwords for every service. For high-risk situations:

```bash
# Recommended password manager setup:
# 1. Bitwarden (self-hosted option available)
# 2. Create NEW master password - never reuse any old password
# 3. Enable 2FA with hardware key (YubiKey) if possible
# 4. Export/import only critical account credentials
# 5. Enable vault timeout (auto-lock after 1 minute)
# 6. DO NOT import full old password database
```

For users uncomfortable with technical tools, even writing passwords on paper (stored securely) is better than reusing passwords across accounts.

### Two-Factor Authentication

Move beyond SMS-based 2FA whenever possible. SIM swapping remains a viable attack:

```bash
# 2FA migration priority:
# 1. Hardware security keys (YubiKey, Titan) - highest security
# 2. Authenticator apps (Bitwarden Authenticator, Aegis for Android)
# 3. Backup codes (print, store in safe location)
# AVOID: SMS verification for any security-critical account
```

For accounts that only support SMS, contact support to see if alternative 2FA methods are available. Some banks and financial institutions can add additional security layers.

## Phone Number Security

### Obtaining a New Number

Phone numbers serve as authentication anchors. Secure the line:

```bash
# New phone number acquisition:
# 1. Obtain new SIM card - pay with cash if possible
# 2. Use new email for account registration
# 3. Enable PIN/PUK code protection on carrier account
# 4. Request port-out protection/porting freeze
# 5. Enable carrier-level call screening
```

### Removing Number from Accounts

Audit accounts linked to the old number:

- Financial services (banks, PayPal, Venmo)
- Social media accounts
- Shopping sites with saved payment methods
- Government services (IRS, DMV)
- Healthcare portals

Create a systematic checklist and methodically work through each service.

## Secure Communications

### Messaging Platform Selection

Signal provides the best balance of security and usability:

```bash
# Signal hardening for high-risk users:
# 1. Enable registration lock (requires PIN to reregister)
# 2. Set disappearing messages for ALL conversations
# 3. Disable notification previews in system settings
# 4. Enable screen lock within Signal settings
# 5. Turn off link previews (prevents metadata leakage)
# 6. Use Screen Security to block screenshots
```

For users needing anonymity, consider:
- VoIP numbers from Twilio or similar (registered to new email)
- Signal with registered VoIP number instead of primary
- Burner phones for highest-risk communications

### Emergency Communication Plan

Establish communication plans before they're needed:

```bash
# Pre-arranged safety protocol:
# 1. Code word that indicates danger/distress
# 2. Trusted contact who knows the situation
# 3. Check-in schedule (if missed, contact initiates welfare check)
# 4. Physical safe location information
# 5. Offline communication method (written notes)
```

Document this plan in a secure location—not on devices that might be compromised.

## Data Protection

### Photo and File Security

Photos often contain location metadata. Before transferring any files:

```bash
# Strip EXIF data before sharing/saving:
# exiftool -all= -overwrite_original image.jpg
# Or use ImageMagick:
# convert input.jpg -strip output.jpg
```

For sensitive documents:

```python
# Python example: Encrypt sensitive files before storage
from cryptography.fernet import Fernet

# Generate key (store securely, separately from files)
key = Fernet.generate_key()
f = Fernet(key)

# Encrypt file
with open('document.pdf', 'rb') as file:
    encrypted = f.encrypt(file.read())

with open('document.pdf.enc', 'wb') as file:
    file.write(encrypted)
```

### Cloud Account Audit

Review all cloud storage for sensitive data:

```bash
# Cloud service audit checklist:
# 1. Dropbox, Google Drive, OneDrive - remove old devices
# 2. iCloud/Google backup - disable if not needed
# 3. Photo sharing - revoke access to ex-partner
# 4. Shared calendars - remove from ex-partner's accounts
# 5. Family sharing schemes - exit all shared arrangements
```

## Ongoing Maintenance

Security isn't a one-time configuration. Establish regular review habits:

- Weekly: Check for unknown devices on accounts
- Monthly: Review privacy settings on social media
- Quarterly: Update passwords on critical accounts
- After any concerning incident: Immediate password change on potentially compromised accounts

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
