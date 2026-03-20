---
layout: default
title: "How to Create a New Digital Identity After Escaping Domestic Violence"
description: "A technical guide for developers and power users on creating a new digital identity after escaping domestic violence. Covers secure email, phone separation, password managers, and operational security."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-create-new-digital-identity-after-escaping-domestic-v/
categories: [guides]
reviewed: true
score: 8
---

{% raw %}

# How to Create a New Digital Identity After Escaping Domestic Violence

Creating a new digital identity requires more than just opening new accounts. For survivors of domestic violence, every digital footprint can become a vector for tracking, harassment, or coercion. This guide provides technical strategies for building an isolated, secure digital presence using tools and workflows accessible to developers and power users.

## Understanding the Threat Model

Before implementing any changes, identify what you're protecting against. In domestic violence scenarios, adversaries often have:

- Physical access to your devices
- Knowledge of your passwords, PINs, and security questions
- Access to shared accounts (phone plans, streaming services, cloud storage)
- Social engineering use through mutual contacts
- Legal use through joint accounts or shared property

Your new digital identity must resist all of these vectors simultaneously.

## Step 1: Secure Communication Channels

### Email Isolation

Create a completely new email address using a privacy-respecting provider. Avoid using any personal information in the address itself:

```bash
# Generate a random email-friendly string
openssl rand -base64 12 | tr -dc 'a-z0-9' | cut -c1-16
# Example output: cGxhY2Vob2xkZX
```

Use this email exclusively for your new identity. Enable two-factor authentication using a hardware token like YubiKey or a TOTP authenticator stored on a new device.

### Encrypted Messaging

For sensitive communications, use end-to-end encrypted messaging applications:

```bash
# Install Signal on Linux (if available)
flatpak install flathub org.signal.Signal
```

Signal provides disappearing messages, screen shot protection, and registration locking. Enable the registration lock PIN and store it separately from your primary recovery information.

## Step 2: Phone Number Separation

Your phone number is one of the most dangerous linkable identifiers. A stalker with access to your old number can:

- Track your location through carrier triangulation
- Reset passwords on numerous services
- Harass you through SMS and calls
- Access your call logs through social engineering

Consider these approaches:

### VOIP with No Carrier Association

Services like MySudo, Google Voice, or dedicated VoIP providers can provide numbers not tied to your physical SIM. However, understand the tradeoffs:

| Method | Pros | Cons |
|--------|------|------|
| Google Voice | Free, integrates with Android | Requires Google account |
| MySudo | Multiple identities, no SIM | Costs money, US-based |
| Burner apps | Temporary numbers | Not reliable for 2FA |

For maximum security, purchase a prepaid SIM card with cash from a location separate from your home address. Use this SIM only for critical accounts.

## Step 3: Password Manager Migration

A password manager becomes your new security backbone. For survivors, the迁移 (migration) process requires extra precautions:

### Creating a New Master Password

Generate a high-entropy master password:

```bash
# Generate 25-word passphrase using EFF wordlist
shuf -n 5 /usr/share/dict/words | tr '\n' '-' | sed 's/-$//'
# Or use bitwarden's generator via CLI
bw generate --length 24 --includeNumber --includeSymbol
```

Store this master password in multiple secure locations: a physical safe, trusted family member's secure storage, and optionally a separate password manager on a secure device.

### New Vault Setup

Initialize a fresh vault in your password manager:

```bash
# Bitwarden CLI creating a new login
bw login
# Create a new folder for critical accounts
bw folder create "critical-recovery"
```

Start by adding only essential accounts. Resist the urge to import your old vault—manual entry forces you to evaluate each account's necessity and security.

## Step 4: Device Hardening

Your devices carry the weight of your digital identity. Securing them means controlling both physical and logical access.

### Factory Reset Protocol

Before resetting any device you'll continue using:

1. Remove any accounts linked to the device
2. Remove from Find My Device services (Apple ID, Google Find My Device)
3. Decouple from smart home systems
4. Document any data you genuinely need to preserve (and encrypt it separately)

A clean factory reset removes firmware-level compromises but creates a fresh attack surface. Reinstall only necessary applications from official sources.

### Device Encryption

Ensure full disk encryption is enabled:

```bash
# Linux: check LUKS status
cryptsetup luksDump /dev/sda1

# macOS: enable FileVault
sudo fdesetup enable

# Windows: enable BitLocker
manage-bde -on C:
```

## Step 5: Account Creation Hygiene

When creating new accounts, follow strict operational security:

### Username Strategy

Avoid reusing usernames across platforms. Generate unique identifiers:

```bash
# Generate a unique username
echo "user$(date +%s | sha256sum | head -c 8)"
```

Many services allow you to change display names without changing usernames—use this to your advantage.

### Security Questions

Never provide truthful answers to security questions. Use your password manager to generate and store random strings:

```
Question: Mother's maiden name
Answer: xK9#mP2$vL8@nQ4
```

Store these as secure notes in your password manager.

## Step 6: Financial Separation

Financial accounts require particular attention. Start with:

1. Opening a new bank account at a different institution
2. Requesting a new debit card with a new PIN
3. Updating direct deposit information with your employer
4. Monitoring credit reports for unauthorized activity

Consider credit freezes with all three bureaus (Equifax, Experian, TransUnion) to prevent identity theft.

## Ongoing Operational Security

Creating a new digital identity is not a one-time event—it's a practice:

- **Audit app permissions monthly**: Review what data your applications access
- **Rotate credentials quarterly**: Change passwords and API keys on a schedule
- **Check for data broker listings**: Use services like DeleteMe to remove your information
- **Maintain physical security**: Device security is meaningless if someone physically accesses them

## Recovery Contacts

Establish a trusted circle who can help in emergencies. Provide them with:

- A sealed envelope containing your master password (stored securely)
- Instructions for account recovery
- Emergency contact information

This redundancy protects against losing access to your own accounts.

Building a new digital identity after escaping domestic violence requires deliberate action across multiple domains. Each step reduces the attack surface available to an adversary. Start with the highest-risk vectors—phone and email—and work downward. The process takes time, but the security gained provides lasting protection.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
