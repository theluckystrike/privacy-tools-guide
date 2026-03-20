---
layout: default
title: "Communicate with Lawyer Privately When Device is Compromised"
description: "A practical guide for individuals who need to maintain attorney-client privilege while dealing with a compromised device. Learn secure communication methods."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-communicate-with-lawyer-privately-when-device-compromised/
categories: [guides, security]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

When your device has been compromised—whether by malware, theft, or unauthorized access—communicating with your attorney requires extra precautions. Standard email and messaging platforms may expose sensitive discussions to whoever controls your compromised device. This guide covers practical methods to maintain attorney-client privilege and keep your legal communications private even when your primary device is not secure.

## Why Standard Communication Methods Fail After Device Compromise

A compromised device grants attackers varying levels of access depending on the type of compromise. Keyloggers can capture everything you type, including passwords and message content. Screen recorders may capture sensitive documents you view. Some malware provides full remote access, allowing attackers to read your communications in real time before you even send them.

Email presents particular risks because unencrypted messages can be intercepted at multiple points. Standard email providers store messages on servers that may be subpoenaed or hacked. Even encrypted email leaves metadata—information about who you contacted, when, and how often—which can be valuable to adversaries.

## Immediate Actions Before Communicating with Your Attorney

Before reaching out to your lawyer, take these critical steps to prevent further exposure:

1. **Disconnect the compromised device from the internet** if you suspect active remote access malware
2. **Avoid logging into any accounts** from the compromised device, including email
3. **Use a different, trusted device** for all sensitive communications
4. **Change passwords** for email and messaging accounts from a clean device
5. **Enable two-factor authentication** on all important accounts using a new authentication method

If you have no access to a clean device, visit a library computer, a friend's phone, or a public computer. Document what happened and preserve evidence if the compromise relates to legal matters.

## Secure Email Options for Attorney Communication

### Proton Mail

Proton Mail provides end-to-end encryption where only you and your recipient can read messages. Even Proton cannot access your email content. For attorney communication:

1. Create a new Proton Mail account from a clean device
2. Share your public key with your attorney (they must also use Proton Mail or import your key)
3. Compose messages using the encryption feature
4. Set expiration times on sensitive messages

Proton's zero-access encryption means that even if legal requests are made, the provider cannot disclose message content—only metadata.

### Secure Email with PGP Encryption

For technically inclined users, PGP encryption provides the strongest email security. Both parties need to set up PGP keys:

```bash
# Generate a new PGP key pair
gpg --full-generate-key

# Export your public key
gpg --armor --export your@email.com > mypublickey.asc

# Encrypt a message for your attorney
gpg --encrypt --armor --recipient attorney@lawfirm.com --output encrypted_message.asc message.txt
```

Share your public key securely—ideally in person or through a verified channel. Your attorney will need their own PGP setup to reply securely.

## Encrypted Messaging Alternatives

### Signal

Signal provides end-to-end encryption by default for text messages, voice calls, and video calls. However, phone number registration creates metadata concerns. For sensitive legal matters:

1. Install Signal on a clean device
2. Register with a dedicated number not linked to your primary identity
3. Enable disappearing messages with short expiration times
4. Verify your attorney's safety number in person or through a separate verified channel

Remember that while message content is encrypted, Signal collects metadata including who you contacted and when.

### Session

Session offers messenger-style communication without requiring a phone number. This eliminates the metadata link between your identity and your communications:

1. Install Session from the official website
2. Generate a Session ID that functions as your identifier
3. Share this ID with your attorney through a separate channel
4. Messages route through decentralized nodes, providing strong anonymity

Session does not collect phone numbers or require personal information, making it suitable for high-sensitivity communication.

## Secure File Sharing for Legal Documents

When sharing sensitive documents with your attorney, avoid standard file sharing services:

### Onion Share

Onion Share lets you share files directly through the Tor network:

1. Install Onion Share from the official site
2. Select "Share Files"
3. The software generates a unique .onion URL valid for one download or a specified time
4. Share this URL with your attorney through a separate channel
5. The file transfers directly between computers without centralized servers

This approach leaves no metadata on third-party servers.

### Tresorit

Tresorit provides encrypted cloud storage with end-to-end encryption:

1. Create a tresor (encrypted folder)
2. Invite your attorney to the tresor
3. Files encrypt locally before upload—tresorit cannot access content
4. You can revoke access at any time

This service works well for ongoing legal matters requiring document sharing.

## Physical Methods for Maximum Security

For the most sensitive communications, consider methods that don't involve digital transmission:

### Paper Documents

Print documents on a clean printer—ideally one you control. Hand-deliver documents to your attorney's office when possible. If mailing, use a postal service that doesn't require identification and consider anonymous drop points.

### Dead Drops

For ongoing anonymous communication, establish a dead drop location where both parties can exchange documents without meeting. Use a location with foot traffic that doesn't require surveillance footage review.

### In-Person Meetings

Whenever possible, meet your attorney in person. Choose locations without surveillance cameras if possible. Turn off phones before the meeting and leave them in your car or a faraday bag.

## What to Avoid

Several common communication methods pose unacceptable risks after device compromise:

- **Standard SMS/MMS**: No encryption, easily intercepted
- **Regular email**: Often unencrypted, leaves extensive metadata
- **Cloud storage links**: Unless end-to-end encrypted before upload
- **Video calls on standard platforms**: Can be recorded, metadata collected
- **Messaging apps requiring phone number linkage**: Metadata exposes relationships

## Preserving Attorney-Client Privilege

Beyond secure communication channels, protect privilege through careful practices:

1. **Clearly label communications** as "Attorney-Client Privileged"
2. **Use dedicated devices** for legal matters when possible
3. **Avoid discussing privileged matters** on any device you don't control
4. **Document your security measures** so privilege isn't waived
5. **Confirm your attorney's security practices** before sharing sensitive information

Your attorney should understand device compromise scenarios and have their own recommendations. Many law firms now offer secure client portals specifically designed for sensitive communications.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [How to Set Up Encrypted Communication for Mutual Aid Network](/privacy-tools-guide/how-to-set-up-encrypted-communication-for-mutual-aid-network/)
- [Best Encrypted Email Service 2026](/privacy-tools-guide/best-encrypted-email-service-2026/)
- [How to Use Signal Disappearing Messages Best Practices](/privacy-tools-guide/signal-disappearing-messages-best-practices/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
