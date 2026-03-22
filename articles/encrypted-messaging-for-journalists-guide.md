---
layout: default
title: "Encrypted Messaging for Journalists Guide"
description: "Signal, Briar, and SecureDrop configured for journalist-source communication. Metadata protection, device hygiene, and air-gap setup covered."
date: 2026-03-22
author: theluckystrike
permalink: /encrypted-messaging-for-journalists-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

# Encrypted Messaging for Journalists Guide

Journalist-source communication faces a specific threat model: state actors, corporate investigators, and law enforcement who may have access to carrier metadata, subpoena powers, and device seizure capabilities. Encrypting the message content is the minimum. Protecting metadata — who messaged whom, when, how often — often matters more.
---

## Key Takeaways

- **Create account → username**: only (no phone, no email) 2.
- **Registration - Use a**: number that isn't publicly linked to you - Options: Google Voice, MySudo, prepaid SIM (cash-purchased) - Don't use your work or personal number 2.
- **Scanning a QR code**: in person (most secure) 2.
- **Signal's sealed sender encrypts the sender's identity**: Signal servers can verify that the message came from a Signal user but not which one.

## Threat Model for Journalists

Before choosing tools, be specific about the threat:

| Threat | Most likely for |
|--------|----------------|
| Content of messages | Sources sharing internal documents |
| Metadata (who communicated with whom, when) | Sources in sensitive positions |
| Physical device seizure | Journalists in hostile jurisdictions |
| Carrier-level surveillance | Domestic sources in high-surveillance countries |
| Network adversary (ISP, national firewall) | Cross-border source communication |

Content encryption (Signal, PGP) addresses the first. Metadata protection requires more: anonymized identifiers, Tor, or air-gapped physical drops.

---

## Signal: Best Default Choice

Signal provides:
- End-to-end encryption (Signal Protocol, open standard)
- Minimal metadata retention (sealed sender, sender/recipient not visible to Signal servers)
- Disappearing messages
- Note-to-self for secure storage
- Screen security (prevents screenshots)

```
Signal setup for journalists:

1. Registration
   - Use a number that isn't publicly linked to you
   - Options: Google Voice, MySudo, prepaid SIM (cash-purchased)
   - Don't use your work or personal number

2. Settings → Privacy:
   - Screen Lock: ON
   - Screen Security: ON (prevents screenshots)
   - Incognito Keyboard: ON
   - Note Reactions: OFF (reduces metadata)

3. Settings → Privacy → Phone Number:
   - "Who can see my phone number": Nobody
   - "Who can find me by number": Nobody

4. Settings → Notifications:
   - "Show": No name or message
   - This prevents lock screen leaking message content

5. Enable Registration Lock:
   - Settings → Account → Registration Lock → ON
   - This prevents SIM swap attacks from taking over your Signal
```

**Disappearing messages default:**

```
Signal Settings → Privacy → Default Timer for New Chats → 1 week

Individual conversation: press timer icon → set per your source's needs
For high-risk sources: 1 day or 1 hour
```

---

## Signal's Sealed Sender

Standard messaging reveals sender identity to the service provider (even for E2EE). Signal's sealed sender encrypts the sender's identity — Signal servers can verify that the message came from a Signal user but not which one.

```
Enable for all contacts:
Settings → Privacy → Advanced → Sealed Sender → Allow from anyone
```

This is enabled by default for existing contacts. "Allow from anyone" lets sources message you anonymously even without being in your contacts.

---

## Briar: Peer-to-Peer, Works Without Internet

Briar is a messaging app that routes messages over Tor and can work over Bluetooth or WiFi when internet is unavailable — useful in situations where network surveillance is likely or connectivity is cut.

```bash
# Install Briar
# Android: F-Droid → search Briar, or:
# https://briarproject.org/apk/briar.apk

# Desktop (Linux/Windows/macOS):
# Download from briarproject.org
wget https://desktop.briarproject.org/linux64/briar-desktop-linux-x64.AppImage
chmod +x briar-desktop-linux-x64.AppImage
./briar-desktop-linux-x64.AppImage
```

**Briar's key advantage over Signal**: No phone number required. Add contacts by:
1. Scanning a QR code in person (most secure)
2. Exchanging a link over any channel
3. Physical proximity (Bluetooth/WiFi if no internet)

```
Briar setup:
1. Create account → username only (no phone, no email)
2. Share your Briar link with sources (long alphanumeric string)
3. Messages route over Tor by default
4. Enable: Settings → Privacy → require all connections via Tor
```

---

## SecureDrop: Whistleblower Submission System

SecureDrop is for initial contact from anonymous sources, not ongoing conversation. Sources access it via Tor Browser — they never reveal their identity.

```bash
# SecureDrop requires a dedicated server
# Minimum: two air-gapped servers (App Server + Monitor Server)
# Full installation guide: docs.securedrop.org

# For journalists using an existing SecureDrop installation:
# Your organization's SecureDrop .onion address is published on your website
# Sources access it via Tor Browser
# You check submissions via Secure Viewing Station (air-gapped laptop)

# Check if a media organization has SecureDrop:
# https://securedrop.org/directory/
```

**For individual journalists without institutional SecureDrop**, use OnionShare:

```bash
# OnionShare can receive files anonymously over Tor
# Source needs Tor Browser only

onionshare --receive
# Creates an .onion address
# Share this address with source via Signal/Briar
# Source uploads files at the .onion address via Tor Browser
# You receive files without the source revealing identity
```

---

## PGP for Email

When sources prefer email, PGP provides content encryption. Metadata (sender, recipient, timestamp) remains visible to email providers.

```bash
# Generate a PGP key pair
gpg --full-generate-key
# Choose: RSA 4096, expires 2 years

# Export public key for sources
gpg --armor --export you@email.com > public-key.asc

# Publish your public key:
# 1. Your publication's website
# 2. keys.openpgp.org
gpg --keyserver keys.openpgp.org --send-keys YOUR_KEY_ID

# Receive encrypted email:
gpg --decrypt encrypted-message.asc

# Verify a signed message
gpg --verify message.asc.sig message.asc
```

**Thunderbird + Enigmail/built-in OpenPGP:**

```
Thunderbird → Settings → End-to-End Encryption
Add Key → use existing key or generate new
Enable encryption by default for new messages
```

---

## Device Hygiene for Source Communications

### Separate Device

The strongest approach: dedicate a device (phone or laptop) exclusively to source communication. This device:

- Never logs into personal accounts
- Uses a non-personal SIM or WiFi only
- Boots Tails OS (amnesic — nothing persists)
- Has full-disk encryption enabled

```bash
# Boot Tails from USB for sensitive source communication
# All data is in RAM and wiped on shutdown

# Tails includes Signal Desktop, Tor Browser, OnionShare
# Set up Signal with a temporary number for this session
```

### Signal's Note to Self + Disappearing Messages

```
Signal → Note to Self (your own contact)
Set disappearing messages timer

Use this to store source-provided credentials or documents temporarily
Data is E2EE on Signal's servers under your key
Set timer so it auto-deletes
```

---

## Metadata Protection: What Each Tool Leaks

| Tool | Content | Who you talked to | When | How often |
|------|---------|------------------|------|----------|
| SMS | To provider | To provider | To provider | To provider |
| WhatsApp | E2EE | To Meta | To Meta | To Meta |
| Signal | E2EE | Minimal (phone #) | Some | Some |
| Briar (Tor) | E2EE | Nothing | Nothing | Nothing |
| SecureDrop | E2EE | Nothing | Nothing | Nothing |
| PGP email | Content only | To email provider | To email provider | To email provider |

For high-risk sources: Briar or SecureDrop are the only options that approach true metadata protection.

---

## Legal Considerations

In the US, journalists may have limited shield law protection for source identity — but this varies by state and is not absolute. Federal investigations often override state shield laws.

```
Practical implications:
- If your device is seized, what does it reveal?
  → Test: open Signal, what's visible without the PIN?
  → Enable Screen Security and PIN lock

- If Signal/Briar is compelled to turn over data:
  → Signal has almost nothing to give (metadata only)
  → Briar has nothing on servers (peer-to-peer)

- If a carrier is subpoenaed for call records:
  → Signal over WiFi leaves no carrier metadata
  → SMS and voice calls are fully logged by carriers
```

---

## Related Reading

- [Best Encrypted Messaging for Journalists](/privacy-tools-guide/best-encrypted-messaging-for-journalists/)
- [How to Set Up Encrypted Dead Drop with OnionShare](/privacy-tools-guide/how-to-set-up-encrypted-dead-drop-using-onionshare-for-sourc/)
- [Anonymous Email Over Tor Setup Guide](/privacy-tools-guide/anonymous-email-over-tor-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
