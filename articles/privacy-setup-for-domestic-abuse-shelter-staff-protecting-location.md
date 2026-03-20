---
layout: default
title: "Privacy Setup for Domestic Abuse Shelter Staff: Protecting Location Data"
description: "A technical guide for shelter staff and developers building privacy-respecting tools. Learn how to configure devices, networks, and applications to protect survivor location data."
date: 2026-03-16
author: theluckystrike
permalink: /privacy-setup-for-domestic-abuse-shelter-staff-protecting-location/
categories: [privacy, security, guides]
reviewed: true
intent-checked: true
voice-checked: true
---

{% raw %}

Location data represents one of the most critical privacy vulnerabilities for domestic abuse survivors. When a survivor accesses support services, their device can inadvertently reveal their physical location through dozens of data points: GPS coordinates, cell tower connections, WiFi networks, app telemetry, and metadata. For shelter staff and developers building privacy-respecting tools, understanding how to minimize this exposure is essential.

This guide provides practical, technical approaches to protecting location privacy. The strategies work whether you're hardening your own devices or architecting systems for an organization serving survivors.

## Understanding Location Data Exposure

Modern smartphones collect location data through multiple mechanisms. The most obvious is GPS, which provides precise coordinates. However, several other data sources can reveal location even when GPS is disabled:

- **Cell tower IDs**: Your phone connects to nearby cell towers, and each tower has a unique identifier that can be mapped to a geographic area
- **WiFi BSSIDs**: Networks your device has connected to can be geolocated based on their registered positions
- **Bluetooth beacons**: Nearby Bluetooth devices can serve as location proxies
- **App and browser APIs**: Many applications request location permissions or infer location from IP addresses
- **Metadata**: Timestamps, network logs, and other metadata can correlate with movement patterns

For shelter staff, the threat model includes abusive partners who may have technical expertise or access to commercial surveillance tools. This means defense-in-depth matters.

## Device Hardening for Shelter Staff

### Mobile Device Configuration

On iOS, navigate to **Settings > Privacy & Security > Location Services** and review which apps have access. Consider enabling "While Using" instead of "Always" for most apps. For critical safety apps, you may want to disable location entirely and enter locations manually.

```bash
# iOS: Check which apps have location access via shortcuts
# This creates a list of apps with location permissions
shortcuts run "Get Location Apps"
```

On Android, use the Privacy Dashboard to monitor location access. Android 12+ includes a location indicator that shows when apps access location data. For shelter devices, consider installing a secondary user profile with restricted permissions:

```bash
# Android: Create a restricted profile (requires device admin)
adb shell pm create-user --ephemeral --managed --demo shelter_profile
adb shell pm set-user-restriction android:no_share_location true
```

### Disabling Radios When Not Needed

When devices are in secure locations, disabling wireless radios reduces attack surface:

```bash
# iOS Shortcut to disable location services and wireless
# Create via Shortcuts app - this is the automation approach

# For Linux systems, use rfkill
rfkill block wifi bluetooth
rfkill unblock wifi  # when needed
```

### Network-Level Protection

Shelter networks should implement network segmentation and DNS-based privacy:

```yaml
# Pi-hole configuration for blocklists
# /etc/pihole/adlists.list

# Block known location-tracking domains
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://v.firebog.net/hosts/Prigent-Ads.txt
https://v.firebog.net/hosts/Prigent-Mobile.txt

# Add organization-specific blocks
address=/location.apple.com/0.0.0.0
address=/geoip-api.apple.com/0.0.0.0
```

## Application Configuration for Privacy

### Browser Privacy Settings

Staff browsers should be configured to minimize location leakage:

```javascript
// Firefox about:config privacy settings
user_pref("geo.enabled", false);
user_pref("geo.provider.network.url", "");
user_pref("privacy.resistFingerprinting", true);
user_pref("network.cookie.cookieBehavior", 1);
```

For maximum privacy, consider running a dedicated browser profile with these settings applied through a policy file:

```json
// Firefox autoconfig (policies.json)
{
  "policies": {
    "Privacy": {
      "DisableLocation": true,
      "ResistFingerprinting": true,
      "ClearOnExit": ["cache", "cookies", "history", "formdata"]
    }
  }
}
```

### Messaging and Communication

Staff should use end-to-end encrypted messaging with ephemeral messages enabled. For sensitive communications, consider:

```bash
# Signal configuration (manual settings within app)
# Enable "Disappearing Messages" for all conversations
# Set timer to 5 minutes for sensitive discussions
# Disable link previews to prevent URL leakage
# Enable "Screen Security" to prevent screenshots
```

### Email Hardening

Email headers often contain location metadata. Configure email clients to strip headers:

```python
# Python script to sanitize email headers
import email
from email import policy
from email.parser import BytesParser

def sanitize_email(raw_email):
    msg = BytesParser(policy=policy.default).parsebytes(raw_email)
    
    # Remove location-related headers
    headers_to_remove = [
        'X-Originating-IP',
        'X-Sender-IP',
        'Received',
        'X-Gm-Message-State'
    ]
    
    for header in headers_to_remove:
        if header in msg:
            del msg[header]
    
    return msg

# Usage
with open('suspicious_email.eml', 'rb') as f:
    cleaned = sanitize_email(f.read())
```

## Network Architecture for Shelters

For organizations, network architecture plays a critical role in protecting location privacy:

### VPN Implementation

Staff working remotely should use organization-provided VPNs that route traffic through secure endpoints:

```yaml
# WireGuard server configuration example
[Interface]
PrivateKey = <server-private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.0.0.2/32
PersistentKeepalive = 25
```

### Captive Portal Isolation

Shelter guest networks should implement client isolation to prevent device discovery:

```bash
# Linux bridge with client isolation
# /etc/network/interfaces
auto br0
iface br0 inet static
    bridge-ports eth0 eth1
    bridgefd 0
    bridge-hello 0
    bridge-maxage 0
    bridge-stp disabled
```

## Data Handling Best Practices

### Minimizing Data Collection

Organizations should adopt data minimization principles:

```python
# Example: Location data anonymization
import hashlib
import secrets

def anonymize_location_data(lat: float, lon: float, salt: bytes = None) -> str:
    """
    Round coordinates to reduce precision,
    then hash to prevent reversal.
    """
    # Round to ~1km precision
    lat_rounded = round(lat, 2)
    lon_rounded = round(lon, 2)
    
    # Generate salt if not provided
    if salt is None:
        salt = secrets.token_bytes(32)
    
    # Create deterministic but non-reversible identifier
    data = f"{lat_rounded},{lon_rounded}".encode()
    return hashlib.pbkdf2_hmac('sha256', data, salt, 100000).hex()[:16]
```

### Secure Data Storage

When location data must be stored, implement encryption at rest:

```bash
# Linux: Create encrypted container for sensitive data
# Using LUKS
cryptsetup luksFormat /dev/sdb1
cryptsetup open /dev/sdb1 secure_volume
mkfs.ext4 /dev/mapper/secure_volume
mount /dev/mapper/secure_volume /mnt/secure_data
```

## Incident Response Considerations

Staff should have clear protocols for suspected surveillance:

1. **Device inspection**: Have a prepared list of apps installed for comparison
2. **Network monitoring**: Log connected devices on shelter networks
3. **Communication backup**: Ensure critical contacts exist outside primary channels
4. **Physical security**: Consider device storage in Faraday bags during sensitive meetings

## Continuous Improvement

Privacy configuration requires ongoing attention. Establish a quarterly review process:

- Audit installed applications and permissions
- Update blocklists and filter configurations
- Review access logs for anomalies
- Test incident response procedures
- Update staff training on emerging threats

Protecting location privacy for domestic abuse survivors demands technical rigor combined with operational discipline. The strategies in this guide provide a foundation, but each organization's implementation requires careful assessment of their specific threat model and resources.

For developers building tools for this population, prioritize privacy by design: collect only necessary data, minimize retention periods, encrypt everything, and provide users with meaningful controls over their information.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
