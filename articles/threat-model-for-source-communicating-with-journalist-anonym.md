---
layout: default
title: "Threat Model for Source Communicating with Journalist."
description: "A comprehensive threat model guide for sources communicating with journalists anonymously. Covers secure communication tools, operational security, and."
date: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-source-communicating-with-journalist-anonym/
categories: [guides, security, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Understanding how to communicate with journalists securely requires a structured threat model. Unlike general privacy guides, source protection demands attention to metadata, timing attacks, and adversary capabilities that extend beyond typical personal security. This guide provides a practical framework for developers and power users evaluating their threat model when reaching out to journalists anonymously.

## Identifying Your Adversary

The first step in building a threat model involves understanding who might be investigating your communications. For source-journalist communication, adversaries typically include:

- **Nation-state actors** with sophisticated surveillance capabilities
- **Corporate security teams** with legal process power (subpoenas, warrants)
- **ISP-level monitoring** that can correlate traffic patterns
- **Device-level compromises** including keyloggers and firmware attacks

Each adversary brings different capabilities. A nation-state can potentially perform traffic analysis across multiple networks, while a corporation is limited to what they can legally compel or technically observe within their infrastructure. Your threat model should account for the most capable adversary you expect to face.

## Layer 1: Communication Channel Selection

Choosing the right communication platform forms the foundation of your threat model. Not all secure messaging apps provide equivalent protection.

### Signal: The Baseline

Signal provides end-to-end encryption with a focus on reducing metadata. However, it stores contact lists and message timestamps on servers. For sources, Signal's safety number verification prevents man-in-the-middle attacks but requires careful verification.

```bash
# Verify Signal safety numbers in person or via separate secure channel
# On iOS: Settings > Account > Safety Numbers
# On Android: Settings > Account > Safety Numbers
# Compare these numbers through a DIFFERENT channel than Signal
```

Signal's weakness for source protection lies in phone number dependency. Your phone carrier knows your number, and SIM-swapping attacks can compromise accounts. Consider using a dedicated device with a burner SIM for sensitive communications.

### Secure Email: ProtonMail and Tutanota

Encrypted email services provide asynchronous communication but introduce different trade-offs. ProtonMail requires phone number verification for signup, which creates an authentication linkage. Tutanota offers anonymous signup without phone numbers.

```python
# When using encrypted email, verify encryption is actually enabled
# ProtonMail: Settings > Security > Enable encryption
# Always use the recipient's public key for additional encryption layer
# Store public keys separately from the communication channel
```

### Decentralized Options: Session and SimpleX

Session and SimpleX Messenger offer no phone number requirements and distributed infrastructure. Session routes messages through onion-routing similar to Tor, while SimpleX uses a novel mesh network approach without user IDs.

These platforms trade convenience for anonymity. Expect a steeper learning curve and potentially smaller journalist adoption.

## Layer 2: Metadata Protection

Metadata often reveals more than content. Your threat model must address what information leaks even when encryption is perfect.

### Timing Analysis

When you communicate matters as much as what you communicate. Regular communication patterns establish baseline behavior. Sudden spikes in communication around significant events correlate strongly with source activity.

```bash
# Reduce timing correlation by:
# 1. Setting up scheduled, periodic check-ins regardless of content
# 2. Using message delay features in Signal (Settings > Privacy > Reload Messages)
# 3. Sending messages at random times rather than immediately after events
# 4. Batching communications to reduce frequency
```

### Traffic Correlation

If an adversary monitors both your network and the journalist's network, they can correlate communication timing. Using Tor or a VPN obscures your IP address but adds latency and requires trust in the exit node.

```bash
# Using Tor for encrypted messaging:
# 1. Download Tor Browser from torproject.org (verify signatures)
# 2. Use Signal Desktop through Tor (advanced configuration)
# 3. Or use Tor Browser for accessing web-based encrypted email
# Remember: Tor protects IP but timing metadata still exists
```

### Device Fingerprinting

Browser fingerprinting can identify your device even without cookies. Journalists receiving links from sources should open them in Tor Browser or isolated VMs to prevent correlation attacks.

## Layer 3: Operational Security

Technical tools fail without operational discipline. Your threat model must include human factors.

### Device Separation

Consider using a dedicated device for source communications:

- **Burner phone**: Basic phone for Signal, purchased with cash, used only for communication
- **Air-gapped computer**: For handling sensitive documents, never connected to the internet
- **Separate identity**: New email, new phone number, new accounts unrelated to your normal identity

```bash
# Air-gapped document handling workflow:
# 1. Create documents on offline machine
# 2. Transfer via USB to online machine using Tails or similar live OS
# 3. Use professional redacting tools (like Adobe Acrobat redaction)
# 4. Verify no metadata in files (exif, author names, track changes)
# 5. Convert to PDF before transmission
```

### Account Isolation

Create accounts using:

- Email addresses from privacy-respecting providers (ProtonMail, Tutanota)
- Phone numbers from burner SIMs or VoIP numbers (Google Voice, but understand Google ties these to accounts)
- Usernames unrelated to your real identity
- Passwords generated and stored in a password manager, never reused

### Physical Security

Digital security fails without physical security:

- Use screen filters to prevent shoulder surfing
- Enable full-disk encryption on all devices
- Set BIOS/firmware passwords
- Disable Bluetooth and Wi-Fi when not in use
- Consider Faraday bags for sensitive devices

## Layer 4: Document Handling

Documents carry substantial metadata that can identify sources.

### Removing Metadata

```bash
# Using exiftool to strip metadata from images:
exiftool -all= document.jpg
exiftool -all= -overwrite_original document.png

# For PDFs, use pdfinfo to check metadata:
pdfinfo document.pdf

# Remove PDF metadata with pdftk:
pdftk document.pdf dump_data output metadata.txt
# Edit metadata.txt, then:
pdftk document.pdf update_info metadata.txt output document_clean.pdf
```

### Redaction Best Practices

- Never use Word or Google Docs for redacting (metadata persists)
- Use professional redaction tools, not black bars (they can be removed)
- Print and scan documents as images to remove digital traces
- Handwrite sensitive portions, scan as images

## Implementing Your Threat Model

Building a complete threat model requires documenting your decisions:

1. **List your adversaries**: Who might investigate your communications?
2. **Assess their capabilities**: What can they technically and legally accomplish?
3. **Map your attack surface**: What information do you expose through each channel?
4. **Select your layers**: Which tools and practices match your risk tolerance?
5. **Test your setup**: Verify everything works before actual communications
6. **Maintain discipline**: Regular security practices, not just initial setup

Remember that perfect security doesn't exist. The goal is raising the cost of investigation beyond what your adversary is willing to spend. Most sources are compromised through operational failures— reused passwords, unencrypted backups, or social engineering—rather than cryptographic breaks.

Start with Signal as your baseline communication tool, add Tor for network anonymity, separate your source identity from your normal digital life, and build from there based on your specific threat model.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
