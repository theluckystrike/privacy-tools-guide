---
layout: default
title: "Signal Messenger Setup Guide for Journalists: Source."
description: "A practical guide to configuring Signal Messenger for journalists who need secure communications with confidential sources. Covers phone number."
date: 2026-03-20
author: "Privacy Tools Guide"
permalink: /signal-messenger-setup-guide-for-journalists-source-protecti/
categories: [guides]
tags: [privacy-tools-guide]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---


{% raw %}

Signal Messenger provides end-to-end encryption that protects communications from eavesdropping, making it the preferred tool for journalists handling sensitive source communications. While Signal's default settings provide strong security, optimizing the application for source protection requires additional configuration steps. This guide covers the practical setup process for developers and power users who need to maximize their operational security.

## Why Signal for Journalism

Signal uses the Signal Protocol, which provides forward secrecy and deniability properties that distinguish it from typical messaging applications. Forward secrecy ensures that compromise of long-term keys does not expose past conversations. The protocol has undergone extensive cryptographic review and is considered modern for asynchronous messaging.

Journalists face specific threats that generic messaging applications do not adequately address. Sources may be at risk if their contact information becomes known, and message content itself may expose sensitive relationships or information. Signal addresses these concerns through phone number privacy features, sealed sender capabilities, and message expiration controls.

## Initial Setup and Phone Number Privacy

The first security consideration involves phone number visibility. By default, Signal links your account to your phone number, which creates an immediate identification vector. While complete phone number removal requires SMS verification during registration, several privacy settings mitigate exposure.

After installing Signal from the official app store, navigate to Settings > Privacy. The critical settings to enable include:

- **Sealed Sender**: Enable "Allow Sealed Sender deliveries" — this hides metadata about message sender/recipient relationships from Signal servers
- **Typing Indicators**: Disable for sensitive conversations
- **Read Receipts**: Disable to prevent confirmation of message delivery timing
- **Screen Lock**: Enable with a short timeout to prevent physical access

For Android users, the Signal fingerprint verification provides cryptographic assurance of your contact's identity. Compare the 60-character fingerprint through a separate verified channel before discussing sensitive matters.

```bash
# Signal does not expose a CLI for configuration
# All settings must be configured through the mobile application
# However, Signal does support backup export for migration purposes
```

## Disappearing Messages Configuration

Disappearing messages represent a foundational source protection mechanism. When enabled, messages automatically delete from both devices after a configurable duration, reducing the evidence surface if a device is compromised or seized.

Configure disappearing messages per-conversation by tapping the conversation header and selecting "Disappearing Messages". The available durations range from 30 seconds to 1 year. For most source communications, a 24-hour or 7-day setting balances operational needs against the risk of accidental loss.

```yaml
# Signal stores disappearing message settings locally
# The setting is synchronized between conversation participants
# Server does not store message content, only the expiration configuration
```

Important: Disappearing messages only affect new messages after activation. Existing messages remain visible unless manually deleted from both devices. Develop a protocol for source contacts to delete historical conversations before enabling disappearing messages.

## Registration Lock and Account Security

Signal's registration lock feature prevents unauthorized account transfers. When enabled, any attempt to register your number on a new device requires your registration lock PIN, a 6-20 digit number you set during activation.

To enable registration lock:

1. Go to Settings > Account > Registration Lock
2. Enter your chosen PIN and confirm
3. Record this PIN somewhere secure—loss requires account recovery through your phone number

For high-risk scenarios, consider maintaining a secondary device specifically for source communications. This device should:

- Use a dedicated phone number not linked to personal communications
- Run a privacy-focused ROM such as GrapheneOS or CalyxOS
- Remain encrypted at rest with strong device encryption
- Be stored securely when not in use

```bash
# Device encryption verification (Android)
# Settings > Security > Encryption > Encrypt Phone
# Ensure full disk encryption is enabled before handling sensitive conversations
```

## Screen Security and Screenshot Prevention

Signal includes screen security features that prevent screenshots and screen recording on Android. Enable this in Settings > Privacy > Screen Security. This prevents malware on your device from capturing conversation content and blocks unauthorized screen recording.

iOS users should note that screen security is more limited. While Signal attempts to prevent screenshots, the operating system permits screen recording in certain scenarios. Disable screen recording permissions for Signal in iOS Settings > Signal > Screen Recording.

## Managing Contact Discovery

Signal's sync process can inadvertently expose your contacts to Signal's servers. During initial setup, Signal queries your contact list to identify other Signal users. This creates a mapping between phone numbers and Signal accounts that persists on Signal's infrastructure.

To minimize this exposure:

1. During initial sync, choose "Continue Without Syncing" if available
2. Manually add source contacts rather than importing your full contact list
3. Periodically review Settings > Privacy > Contacts to ensure no unexpected syncing

For sources using Signal, recommend they create accounts with dedicated phone numbers that are not publicly associated with their identity. Voice-over-IP numbers from services like Google Voice or dedicated VoIP providers can serve this purpose, though consider that some jurisdictions may require ID verification for VoIP registration.

## Verification and Key Fingerprints

Establishing verification for source communications requires comparing Signal safety numbers. These numbers derive from the cryptographic keys exchanged during the initial conversation, providing cryptographic proof that you communicate with the intended party.

To verify a contact:

1. Open the conversation
2. Tap the contact header
3. Select "View Safety Number"
4. Verify the 60-digit number through a separate channel (in-person, another encrypted messenger, or verified PGP-encrypted email)

Document verified safety numbers for future reference. A change in the safety number indicates potential key compromise or man-in-the-middle attack and should trigger immediate communication through alternate channels to verify the issue.

```python
# Safety number derivation (conceptual)
# Signal combines identity keys and session keys
# The resulting fingerprint is displayed as 60-digit number
# Format: XXX-XXX-XXX-XXX-XXX-XXX-XXX-XXX-XXX-XXX
# Verify all 60 digits through independent channel
```

## Advanced: Note to Self for Secure Storage

Signal's Note to Self feature provides an encrypted scratchpad accessible only from your own devices. While not a replacement for dedicated password managers, this feature offers secure storage for sensitive information like source details, interview notes, or encryption keys.

The Note to Self conversation supports all Signal features including disappearing messages, file attachments, and voice messages. Use this for temporary storage of sensitive information that you would rather not store in cloud-synced notes applications.

## Device Hygiene and Operational Security

Technical configuration alone does not guarantee security. Operational practices complement Signal's encryption:

- **Device storage**: Enable full-disk encryption on all devices
- **Biometric security**: Use fingerprint or face unlock in addition to device PIN
- **Backup practices**: Disable cloud backup for Signal conversations containing sensitive information
- **Notification content**: Configure Settings > Notifications to hide message content in notifications
- **Lock app**: Use Signal's built-in app lock with biometric authentication

```bash
# Hide notification content (Android)
# Settings > Notifications > Notification Content > "Hide"
# This displays "New message" instead of message text in notifications
```

## Advanced: Multi-Device Configuration

For high-risk journalists, maintain multiple phone numbers and devices:

```yaml
# Device 1: Work phone
Phone: +1-555-WORK-001 (public-facing)
Uses: Professional contacts, non-sensitive calls
Signal: Standard configuration

# Device 2: Source phone
Phone: +1-555-SOURCE-001 (private number)
Uses: Source communications only
Signal: Maximum privacy configuration
  - Sealed sender enabled
  - Disappearing messages: 24 hours
  - Screen lock enabled
  - Regular wipe schedule: Every 30 days
  - Backup: Disabled

# Device 3: Backup device
Phone: +1-555-BACKUP-001 (in safe)
Uses: Emergency communications only
Signal: Registered, configured, kept offline
  - Activated only in crisis scenarios
  - Contains backup contacts list
  - Separate registration lock PIN
```

Store backup phone in a safe deposit box or secure location inaccessible to people who might seize devices.

## Jurisdictional Considerations

Different regions apply different laws to encrypted communications:

### United States
- Signal is legal; law enforcement regularly demands chats
- Registration lock PIN can be compelled (though technically difficult)
- Warrant requirements exist but are frequently overridden
- ECPA provides minimal protection for stored messages

### European Union
- GDPR applies; data minimization favored
- Court orders required before Signal content access
- Jurisdiction-specific restrictions on law enforcement access

### Authoritarian Jurisdictions
- Signal may be blocked entirely (check beforehand)
- Use Tor bridges for connectivity if blocked
- Assume all communications may be monitored
- Disappearing messages become critical (evidence destruction)

Research your jurisdiction's laws before handling sensitive sources. Consult with a lawyer familiar with press freedom in your region.

## Technical Verification Against Compromise

If you suspect your device is compromised:

```bash
# Check for known indicators of compromise

# 1. Memory test (advanced)
# If you suspect you have malware, advanced tools can detect it:
chkrootkit (Linux)
rootkit hunter (Linux)
# These may miss sophisticated spyware

# 2. Network monitoring
# Check for unexpected outbound connections
netstat -an | grep ESTABLISHED
lsof -i -P -n

# 3. Check app permissions (Android)
Settings > Apps > [Signal]
- Camera: Should be "Don't allow"
- Location: Should be "Don't allow"
- Microphone: Allowed (required for calls)
```

If you detect compromise, physically destroy the device rather than attempting cleanup. Sophisticated malware is resistant to removal.

## Documentation and Evidence Preservation

Maintain secure documentation of your security practices:

```bash
# Create a security practices document (stored encrypted)
# This protects your privilege claim if source relationship questioned

Document:
- When you established Signal contact with source
- What privacy measures you implemented
- How you verified their identity
- Backup contacts/communication methods
- Date of each significant communication

# Store in encrypted vault alongside Signal conversations
# This demonstrates good faith security practices
```

If questioned by law enforcement or courts about source protection, documentation of your security practices supports attorney-client and source-journalist privilege claims.

## Source Onboarding Process

When bringing a new source onto Signal:

```
Step 1: Provide phone number through verified channel
  - In-person conversation preferred
  - Alternative: Verified phone call (not Signal yet)
  - Never: Email, social media DMs

Step 2: Source installs Signal from official app store
  - iOS: App Store (not TestFlight)
  - Android: Google Play or signal.org

Step 3: You and source verify phone numbers
  - This prevents man-in-the-middle registration

Step 4: Compare safety numbers in person
  - Bring QR code to in-person meeting
  - Source scans QR code on their device
  - Confirms safety number matches

Step 5: Establish communication protocols
  - Disappearing message duration (24 hours recommended)
  - Response time expectations
  - Emergency contact procedures
  - When to stop using Signal (escalation)
```

This process takes 30-45 minutes but provides strong verification that you're actually communicating with your intended source.

## Signal vs Alternatives for Journalists

| Messenger | Encryption | Metadata | Logging | Best For |
|-----------|-----------|----------|---------|----------|
| Signal | E2E | Phone number | Device only | Source protection, standard use |
| Session | E2E | No phone required | Device only | Maximum anonymity |
| Wire | E2E | Email option | Device only | Multi-device teams |
| Telegram | Optional E2E | Phone number | Cloud stored | Non-sensitive comms |

Signal remains the best balance for journalists. Session provides stronger anonymity if phone number registration is unacceptable risk.

## Red Flags for Source Safety

During Signal communication, watch for:

- Unusual requests to move to less-secure channels
- Pressure to share information quickly without verification
- Multiple sources claiming same story independently (coordination possibility)
- Requests for personal information unrelated to story
- Attempts to establish relationship outside professional context

These patterns suggest potential law enforcement manipulation or coordinated source attack. Consult with your news organization's legal and security teams before proceeding.

## Creating a Journalist Security Policy

Organizations should establish written policies:

```markdown
# [News Org] Source Communication Security Policy

## Signal Requirements
- All source communications use Signal
- Registration lock enabled with strong PIN
- Sealed sender enabled for all conversations
- Disappearing messages: 7 days minimum

## Device Requirements
- iOS 15+ or Android 12+
- Regular security updates enforced
- No jailbreak/root allowed for journalists
- Biometric unlock required

## Verification Process
- All new sources: In-person safety number verification
- Existing sources: Re-verify quarterly
- Documentation stored in secure vault

## Incident Response
- Suspected compromise: Immediately notify security team
- Device theft: Remote wipe via Find My / Find My Device
- Law enforcement approach: Refer to legal team
```

Distribute this policy during newsroom onboarding. Regular security training reinforces these practices.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
