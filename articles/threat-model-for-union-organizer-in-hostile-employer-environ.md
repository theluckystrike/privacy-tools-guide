---
layout: default
title: "Threat Model for Union Organizer in Hostile Employer Environment"
description: "A practical threat model for union organizers facing hostile employer environments. Learn actionable digital security strategies for protecting communications, member lists, and organizing activities."
date: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-union-organizer-in-hostile-employer-environ/
categories: [guides]
---

{% raw %}

Union organizers operating in hostile employer environments face adversaries with significant resources, including corporate security teams, legal departments, and increasingly sophisticated surveillance technology. This threat model provides developers and power users with concrete strategies for protecting organizing activities, member communications, and sensitive data from employer retaliation, union-busting tactics, and digital surveillance.

## The Unique Challenges Facing Union Organizers

Unlike other threat models where the adversary is external, union organizers must contend with an adversary who has legal access to certain workplace systems, direct influence over the organizer's employment, and often substantial financial resources to deploy union-busting consultants and technology.

The threat landscape for union organizers includes:

1. **Workplace surveillance** — employer monitoring of work devices, email systems, and network traffic
2. **Social engineering** — union-busting consultants trained in psychological manipulation
3. **Legal coercion** — retaliation through firing, demotion, or frivolous lawsuits
4. **Digital intrusion** — compromise of personal devices and accounts
5. **Physical surveillance** — following organizers to meetings and private locations

A comprehensive threat model addresses each vector while maintaining the practical ability to coordinate organizing activities among coworkers.

## Device Isolation Strategy

The most critical decision in this threat model involves separating work and personal digital lives. Employers typically have legal authority over work devices and may claim ownership over any data stored on them. This makes device isolation essential.

### Dedicated Organizing Hardware

Use a separate, personal smartphone for all union-related communications. This device should:

- Be purchased with cash or using a privacy-focused payment method
- Not be connected to your employer's WiFi network
- Use a privacy-focused mobile OS such as GrapheneOS or CalyxOS
- Have all unnecessary apps removed

```bash
# Verify device integrity before each organizing meeting
# Check for unknown processes and network connections

# On Android (requires root or adb):
adb shell ps -A | grep -v "system\|android" | head -20

# Check active network connections
adb shell netstat -tn | established
```

Keep this device powered off when not in use and store it separately from work belongings. Consider using a Faraday bag to prevent remote activation.

### Network Segmentation

Never use employer's network for union activities. This applies even to personal devices brought into the workplace.

```bash
# Configure your organizing device to block all non-essential network traffic
# Using simple iptables rules on Android via Termux:

# Block all incoming connections
iptables -P INPUT DROP

# Allow established outgoing connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Only allow specific union-related endpoints
iptables -A OUTPUT -p tcp --dport 443 -d unionnews.example.com -j ACCEPT
```

Use mobile data or a trusted VPN service when organizing outside the workplace. The VPN should be configured to prevent DNS leaks and kill all traffic if the connection drops.

## Encrypted Communication Architecture

Union communications require end-to-end encryption that protects content from both employer surveillance and the communication provider itself. Signal remains the gold standard for secure messaging, but additional hardening improves security.

### Signal Hardening Configuration

Configure Signal with these privacy-critical settings:

```javascript
// Signal privacy settings checklist:
// 1. Registration lock: Enable (requires PIN to register new device)
// 2. Relay calls: Enable (prevents caller ID exposure)
// 3. Typing indicators: Disable for group chats
// 4. Read receipts: Disable entirely
// 5. Screen lock: Enable with reasonable timeout
// 6. Incinerator: Enable (auto-deletes old messages)
```

Create a dedicated Signal identity specifically for organizing work. Do not import existing contacts. This separation prevents accidental exposure and makes device forensics more difficult for adversaries.

### Secure Group Communication

For group coordination, consider using Session messenger, which provides onion-routing infrastructure similar to Tor. Session stores no metadata about group membership or message content on central servers.

```bash
# Session protocol advantages over Signal:
# - No phone number requirement (uses public key addresses)
# - No metadata stored on servers
# - Decentralized infrastructure
# - Onion-routing to relay nodes
```

Use disappearing messages with a timeout that matches your operational tempo. If a device is compromised, the window for exfiltrating message history should be minimal.

## Member List Protection

The most sensitive data in any organizing campaign is the member list. This data, if exposed, enables employer retaliation against specific workers and allows union-busting consultants to target vulnerable individuals.

### Air-Gapped Storage

Store member information on air-gapped devices that never connect to any network:

```bash
# Create encrypted partition for member data
# Using LUKS on a dedicated USB drive

# Wipe and create encrypted container
cryptsetup luksFormat /dev/sdX1

# Open the container
cryptsetup luksOpen /dev/sdX1 union_data

# Create filesystem
mkfs.ext4 /dev/mapper/union_data

# Mount and work
mount /dev/mapper/union_data /mnt/secure
```

This USB drive should be physically secured—stored in a location the employer cannot access, possibly in the custody of a trusted co-worker who is not yourself. Never connect this device to any computer used for other purposes.

### Data Minimization

Collect only essential information. Do not maintain detailed personal information about members unless operationally necessary. A minimal member database might include:

```python
# Example: Minimal member data structure
# Only store what's absolutely necessary

member_record = {
    "unique_id": "hash_of_employee_id",  # Never store actual ID
    "department": "general_area",         # Broad category only
    "support_level": 1-5,                 # Numeric, not descriptive
    "last_contact": "YYYY-MM-DD",         # Date only, no time
}
```

The principle here: if you don't have the data, it cannot be seized, leaked, or coerced from you.

## Physical Security Considerations

Digital threats exist alongside physical ones. Organizers must consider how their devices, documents, and movements can be observed.

### Device Seizure Preparation

Configure all devices with encrypted storage and automatic lockout:

```bash
# Linux: Enable encrypted home directory
ecryptfs-setup-private

# macOS: Enable FileVault
sudo fdesetup enable

# Both: Set aggressive auto-lock
# - Lock screen: 1 minute
# - Require password immediately after sleep
# - Erase data after 10 failed attempts
```

If you face sudden device seizure, full-disk encryption protects data provided the device was powered off or locked. Many jurisdictions do not require suspects to provide passwords, but be aware that some do. Know your legal rights in your jurisdiction.

### Meeting Security

Organizing meetings require physical security planning:

- Rotate meeting locations frequently
- Avoid meeting at venues with obvious surveillance
- Leave personal electronics outside when discussing sensitive matters
- Designate a security observer to watch for unusual vehicles or individuals

## Response Planning

Despite precautions, compromise may occur. Prepare response procedures:

1. **Account compromise**: Immediately change passwords from a clean device, enable two-factor authentication on fresh accounts, notify affected members
2. **Device seizure**: Do not attempt to destroy data unless you can do so without detection; encryption provides legal protection in many jurisdictions
3. **Retaliation threats**: Document everything, contact legal counsel, notify union leadership

Build these response procedures into your organizing plan before you need them. Panic leads to mistakes; preparation enables calm, effective response.

---

Protecting union organizing activities in hostile environments requires layered defenses across digital and physical domains. No single measure is sufficient, but a thoughtful combination of device isolation, encrypted communications, minimal data storage, and physical security dramatically improves your security posture. Adapt these strategies to your specific context, threat level, and resources.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
