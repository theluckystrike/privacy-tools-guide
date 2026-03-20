---

layout: default
title: "Threat Model For Source Communicating With Journalist Anonym"
description: "A threat model guide for sources communicating with journalists anonymously. Learn practical tools, encryption methods, and operational."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /threat-model-for-source-communicating-with-journalist-anonym/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Sources communicating with journalists must assume potential state-level compromise including OS-level exploits, network-level interception, and zero-day vulnerabilities; therefore use Tails Linux booted from USB for all sensitive communications, SecureDrop for anonymous document submission, Signal for messaging if using personal devices, and Tor for any internet access. Use a separate device unconnected to your identity, remove all metadata from documents using Mat2 or ExifTool, and establish out-of-band communication channels for verifying journalist identity. This guide provides a practical threat modeling framework covering adversary types, attack vectors, layered defenses, and operational discipline required for source-journalist relationships.

## Understanding the Threat ecosystem

Sources communicating with journalists encounter adversaries with significant resources. Understanding who might attempt to compromise your communication determines which defenses matter most.

### Primary Threat Actors

**State-level adversaries** possess capabilities that can compromise operating systems, intercept network traffic at infrastructure points, and deploy zero-day exploits. If you face this threat level, assume that any device connected to the internet may be compromised.

**Corporate entities** may use legal pressure, technical surveillance, or social engineering to identify sources. They often work through legal discovery, NSL letters, or direct technical compromise of devices.

**Local adversaries** include anyone who might benefit from identifying you as a source—employers, family members, or local authorities. Their capabilities vary widely but should not be underestimated.

### Attack Vectors to Consider

Your threat model must account for multiple attack surfaces:

- **Device compromise**: Malware, firmware attacks, or physical access to your devices
- **Network surveillance**: ISP-level logging, WiFi interception, or traffic analysis
- **Metadata leakage**: Communication patterns, timestamps, and contact associations
- **Operational security failures**: Behavioral patterns, digital traces, or social engineering
- **Legal pressure**: Subpoenas, warrants, or mandatory data retention

## Building Your Defensive Architecture

### Device Selection and Isolation

The foundation of secure source communication starts with dedicated devices that never touch your normal digital life.

Consider using a dedicated laptop or mobile device purchased with cash, never associated with your identity. This device should have:

- Full disk encryption enabled
- No cloud sync services
- Separate, isolated accounts
- Regular security updates applied manually

For maximum protection, use an air-gapped machine for the most sensitive communications, transferring messages via encrypted USB drives using a procedure like this:

```bash
# Verify USB drive integrity before use
sudo cryptsetup luksFormat /dev/sdX
# Create encrypted container
sudo cryptsetup luksOpen /dev/sdX secure_container
sudo mkfs.ext4 /dev/mapper/secure_container
# Mount and use for file transfer
sudo mount /dev/mapper/secure_container /mnt/secure/
```

### Communication Channel Selection

Choose communication channels based on your threat model:

**Secure messaging apps** like Signal provide excellent forward secrecy and end-to-end encryption. However, they require phone number registration and store contact information on devices. Use a burner phone number registered with cash for maximum privacy.

**Email with PGP encryption** offers asynchronous communication without phone number requirements. The learning curve is higher, but metadata exposure is reduced:

```python
# Example PGP encryption workflow using python-gnupg
from gnupg import GPG

gpg = GPG(gnupghome='/path/to/gpg/home')

# Encrypt message for journalist's public key
public_key_data = open('journalist_pubkey.asc').read()
gpg.import_keys(public_key_data)

encrypted = gpg.encrypt(
    "Your message here",
    recipients=['journalist@example.com'],
    always_trust=True
)

print(str(encrypted))
```

**Secure drop systems** like SecureDrop provide institutionalized protection, but require institutional setup and trust in the organization's infrastructure.

### Network Operational Security

Your network connection reveals significant information about your communication patterns.

Tor browser provides anonymity by routing traffic through multiple relays, obscuring your IP address and preventing simple traffic analysis:

```bash
# Verify Tor is working correctly
# Check your IP appears different
curl -s https://check.torproject.org/api/ip
```

Use Tor for all sensitive communications. Avoid logging into personal accounts or revealing identifying information while using Tor. Consider using Tor in a TAILS live operating system for added protection.

### Metadata Protection

Even with encrypted communications, metadata can reveal your network:

| Metadata Type | Risk Level | Mitigation |
|---------------|------------|------------|
| Timestamps | Medium | Use asynchronous communication |
| IP Addresses | High | Always use Tor |
| Contact Lists | High | Keep minimal, use encryption |
| Message Length | Low | Padding can help |
| Communication Frequency | Medium | Vary communication patterns |

## Operational Security Practices

### Identity Separation

Maintain strict separation between your source identity and normal digital presence:

- **Separate devices**: Never use your source device for personal activities
- **Separate identities**: Create entirely new email addresses, phone numbers, and accounts
- **Separate locations**: Access sensitive communications from unexpected locations when possible
- **Separate behaviors**: Your browsing patterns, typing speed, and timing can fingerprint you

### Communication Discipline

Establish protocols that minimize risk:

1. **Pre-agreed security words**: Establish codes for urgent situations
2. **Scheduled check-ins**: Agree on regular communication windows
3. **Dead drop protocols**: Establish fallback methods if primary channels fail
4. **Automatic message deletion**: Configure messages to delete after reading

### Physical Security

Digital security means nothing without physical security:

- Use screen filters to prevent shoulder surfing
- Access sensitive communications in private locations
- Never leave devices unlocked or unattended
- Consider faraday bags for storing sensitive devices

## Recovery and Incident Response

### If You Suspect Compromise

Have a pre-planned response:

1. **Immediately stop** using compromised devices for sensitive communication
2. **Power down** devices—do not reboot, as startup may trigger evidence destruction
3. **Preserve evidence**: If safe, image the device before any cleanup
4. **Switch to backup channels** using pre-established protocols
5. **Notify your journalist** through a secure fallback method

### Backup Communication Plans

Always maintain redundant communication methods:

- Primary: Signal with burner number
- Secondary: PGP-encrypted email through Tor
- Tertiary: Encrypted dead drop using secure file hosting
- Emergency: Physical dead drop with pre-arranged location

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
