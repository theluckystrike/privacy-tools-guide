---
layout: default
title: "Threat Model For Sex Worker Protecting Real Identity And."
description: "A practical technical guide for developers and power users on building a threat model to protect real identity and physical location for sex workers"
date: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-sex-worker-protecting-real-identity-and-location/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Separate your professional and personal identities completely: use dedicated devices, distinct email addresses, separate phone numbers, and different payment methods for work. Enable VPN/Tor for location protection, use data brokers removal services, compartmentalize devices to prevent cross-contamination, and maintain strict operational security to prevent doxxing and physical threats.

## Threat Landscape Assessment

Before implementing protections, identify what you're defending against:

**Adversaries vary by threat model:**

- **Harassment actors**: Ex-partners, angry clients, or coordinated harassment campaigns seeking to expose your identity
- **Data brokers**: Aggregators compiling personal information sold for investigation purposes
- **Legal discovery**: Subpoenas and court orders seeking to identify individuals
- **Platform risks**: Payment processors, social media companies, and websites that may leak or share user data
- **Physical threats**: Individuals seeking your location for confrontation or violence

The core principle: separate your professional persona entirely from your personal identity. This means different phone numbers, email addresses, payment methods, devices, and physical locations for work activities.

## Identity Separation Architecture

The most critical security measure is creating strict separation between your professional and personal digital lives. This goes beyond using different email addresses—it requires complete device and identity isolation.

### Device Segmentation

Use dedicated devices for work activities:

```bash
# Physical device separation strategy
#
# Device 1: Personal
# - Personal email, banking, social media
# - Home IP address
# - GPS location enabled
# - Standard phone number
#
# Device 2: Professional (Work Only)
# - Dedicated work email (anonymous)
# - VoIP numbers only (Signal, etc.)
# - VPN always connected
# - GPS disabled or spoofed
# - Separate Apple/Google account
```

Never mix personal and professional activities on the same device. A single mistake—like logging into your personal Instagram on your work device—can link your identities.

### Phone Number Isolation

Phone numbers are a primary vector for identity linkage. Use separate numbers for everything:

```python
# Phone number separation strategy
PHONE_CONFIG = {
    "personal": {
        "number": "+1-555-0100-xxxx",  # Your real number
        "usage": ["banking", "family", "personal"],
        "location": "home"
    },
    "professional": {
        "number": None,  # Never give this out
        "voip_apps": ["Signal", "Google Voice"],
        "usage": ["client communication"],
        "never_linked_to_personal": True
    }
}

# Recommended setup:
# - Personal: Your real carrier number (kept private)
# - Work: Google Voice or dedicated VoIP (Signal)
# - Client contact: Burner numbers rotated regularly
```

Never use your personal phone number for professional communications. Services like Google Voice, MySudo, or Burner provide separate numbers that don't link to your identity.

### Email Account Stratification

Create a completely separate email infrastructure for work:

```bash
# Email isolation setup
#
# Personal accounts (linked to real identity):
# - personal.name@gmail.com
# - Used for: banking, government, family
#
# Professional accounts (anonymous):
# - work.pseudonym@protonmail.com
# - Used for: client communication, platform accounts
#
# NEVER cross the streams:
# - Don't email your personal account from work email
# - Don't use work email for personal registrations
# - Don't include personal details in work email signatures
```

Use Proton Mail or Tutanota for work accounts—these services don't require phone verification and provide end-to-end encryption. Register these accounts using your work phone number and VPN, never from your home IP address.

## Location Privacy Implementation

Protecting your physical location requires multiple defensive layers. Location data can be exposed through IP addresses, GPS coordinates, metadata, and behavioral patterns.

### IP Address Protection

Always route work traffic through VPN or Tor:

```bash
# VPN configuration for work activities
#
# Essential settings:
# - Kill switch: enabled (blocks traffic if VPN drops)
# - Protocol: WireGuard or OpenVPN
# - DNS: VPN provider DNS only (prevent DNS leaks)
# - IPv6: disabled or routed through VPN
#
# Testing your protection:
curl ifconfig.me           # Your visible IP without VPN
curl --socks5 127.0.0.1:9050 ifconfig.me  # Through Tor

# Verify DNS leaks:
dig +short myip.opendns.com @resolver1.opendns.com
```

For maximum security, use Whonix or Tails for work activities. These operating systems route all traffic through Tor by default and provide network-level isolation.

### GPS and Metadata Protection

Disable location services on work devices and scrub metadata from shared content:

```python
# Metadata removal script for images
from PIL import Image
import piexif

def remove_location_metadata(image_path):
    """Remove GPS metadata from images before sharing."""
    img = Image.open(image_path)
    
    # Check if image has exif data
    exif_dict = {}
    if "exif" in img.info:
        exif_dict = piexif.load(img.info["exif"])
    
    # Remove GPS IFD (International Fingerprint Directory)
    if "GPS" in exif_dict:
        exif_dict["GPS"] = {}
    
    # Create new exif without GPS data
    exif_bytes = piexif.dump(exif_dict)
    
    # Save to new file
    new_path = image_path.replace(".jpg", "_clean.jpg")
    img.save(new_path, "jpeg", exif=exif_bytes)
    
    return new_path

# Usage
clean_file = remove_location_metadata("client_photo.jpg")
print(f"Clean file saved to: {clean_file}")
```

Always verify photos before sharing: disable location services in camera settings, use metadata removal tools, and consider the background of images for identifying information.

### Address Protection

For shipping physical goods or receiving payments:

```bash
# Address protection strategies
#
# Option 1: Private mailbox (CMRA)
# - Rent a box at USPS or private service
# - Use as shipping address for work
# - Pick up packages with ID
#
# Option 2: Mail forwarding services
# - Services like Traveling Mailbox
# - Scans envelopes, forwards packages
# - Creates layer of separation
#
# Option 3: Virtual mailbox
# - Provides street address (not PO Box)
# - Accepts packages from all carriers
# - Forwards or holds based on preference
```

Never use your home address for work activities. A private mailbox or virtual address provides legal separation and physical safety.

## Communication Security

Client communications require encrypted, ephemeral messaging:

```python
# Secure communication checklist
COMMUNICATION_SECURITY = {
    "messaging": {
        "preferred": ["Signal", "Session", "Threema"],
        "avoid": ["SMS", "Telegram (unless verified)", "WhatsApp"],
        "settings": {
            "disappearing_messages": True,
            "screen_lock": True,
            "notification_content": False
        }
    },
    "calls": {
        "preferred": ["Signal", "Phone (via VoIP number)"],
        "avoid": ["Regular phone calls from personal number"]
    },
    "video": {
        "preferred": ["Signal", "Jitsi (with VPN)"],
        "consider": "Obscuring backgrounds, using VPN"
    }
}

# Signal configuration for maximum privacy
SIGNAL_SETTINGS = {
    "privacy": {
        "read_receipts": False,
        "typing_indicators": False,
        "link_previews": False
    },
    "security": {
        "screen_lock": True,
        "registration_lock": True,
        "incognito_keyboard": True
    }
}
```

Signal provides the best combination of usability and security. Enable disappearing messages, disable typing indicators and read receipts, and use the screen lock feature.

## Payment Privacy

Financial privacy is critical. Payment methods leave trails that can be subpoenaed or exploited:

```bash
# Payment method isolation
#
# Recommended hierarchy (by privacy level):
# 1. Cash (highest privacy)
# 2. Cryptocurrency (high privacy with mixing)
# 3. Prepaid cards (moderate privacy, limited use)
# 4. Platform payments (low privacy, platform holds funds)
#
# Cryptocurrency best practices:
# - Use Monero for maximum privacy
# - If using Bitcoin: always use coin mixer (Wasabi, Samurai)
# - Never reuse addresses
# - Buy peer-to-peer (LocalMonero, Bisq) not exchanges
#
# Prepaid cards:
# - Buy with cash, register with fake info
# - Use only until balance depleted
# - Multiple cards, never link to personal accounts
```

Platforms like OnlyFans, Patreon, and specialized sites keep payment records that can be compelled by legal requests. Consider the privacy implications of each platform.

## Operational Security Checklist

Implement these practices systematically:

```bash
# Daily OPSEC routine
OPSEC_CHECKLIST = """
□ Check VPN connected before any work activity
□ Verify no personal accounts logged on work device
□ Review Signal privacy settings weekly
□ Rotate client contact numbers monthly
□ Check for data breaches (Have I Been Pwned)
□ Review photos for metadata before sharing
□ Verify location services disabled on work device
□ Clear browser history and cookies after sessions
□ Check social media privacy settings
□ Monitor for doxxing attempts (Google alerts)
"""

# Monthly maintenance
MONTHLY_TASKS = """
□ Change email passwords
□ Rotate burner numbers if compromised
□ Check data broker removal status
□ Review connected apps and services
□ Audit browser extensions
□ Test backup restoration
□ Update all software and operating systems
"""
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
