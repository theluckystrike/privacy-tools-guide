---

layout: default
title: "Privacy Setup for Domestic Abuse Shelter Staff."
description: "A technical guide for developers and power users implementing privacy measures for domestic abuse shelter staff. Learn location protection, secure."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /privacy-setup-for-domestic-abuse-shelter-staff-protecting-lo/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Protect shelter staff and survivor locations using full-disk encryption on all devices, separate work phones without location services, Signal-encrypted communications, and databases that redact precise addresses while retaining only essential case information. Implement access controls limiting staff visibility to only their assigned cases, disable metadata from photos before sharing, and use Tor for any external communications about residents. This guide covers comprehensive technical implementations for protecting location data, securing communications, and maintaining operational security in shelter management systems.

## Understanding the Threat Model

Location exposure in domestic abuse contexts can have severe consequences. Abusers often possess technical knowledge and may attempt to:

- Track victims through shared device location services
- Monitor shelter locations through public records or social engineering
- Intercept communications containing location information
- Use metadata from photos or messages to determine shelter areas

Shelter staff need defense in depth. No single measure provides complete protection. The goal is raising the cost of surveillance beyond what most adversaries can sustain.

## Device Hardening for Staff

### Mobile Device Configuration

Staff devices should disable location services by default and enable only when necessary. On iOS, use Shortcuts to create toggle shortcuts for quick enable/disable:

```bash
# iOS Shortcut automation concept
# When leaving shelter premises: disable location
# When arriving at shelter: enable for navigation only
```

For Android, Tasker or Locale apps can automate similar behavior based on geofences around the shelter. The key principle: location services should be off during sensitive communications.

### Removing Tracking Vectors

Staff devices require careful audit:

1. **Remove Find My device sharing** — Any shared family Apple ID or Google account with location sharing enabled compromises the device
2. **Disable automatic WiFi logging** — Turn off "Connect to Known Networks" and avoid storing shelter network names
3. **Review app permissions** — Many apps request unnecessary location access
4. **Use browser privacy mode** — Staff browsing should use containers or private windows

```bash
# Firefox container setup for sensitive browsing
# Create separate containers for:
# - Shelter operations (work)
# - Personal activities (personal)
# Never mix the two in same session
```

## Network-Level Protection

### DNS Configuration

Staff networks should use privacy-respecting DNS to prevent query logging:

```bash
# macOS DNS configuration
networksetup -setdnsservers Wi-Fi 1.1.1.1 1.0.0.1
# Or use encrypted DNS
networksetup -setdnsservers Wi-Fi 1.1.1.1 1.0.0.1
# For DoH (DNS over HTTPS) - configure in System Preferences
```

Consider running a local DNS resolver like dnscrypt-proxy for encrypted upstream queries.

### VPN Implementation

Staff working remotely require VPN access to shelter systems. Self-hosted solutions using WireGuard provide better privacy properties than commercial alternatives:

```bash
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

WireGuard's minimal codebase makes auditing easier than OpenVPN alternatives.

## Secure Communications

### Encrypted Messaging

Signal remains the gold standard for encrypted communications. Configure it properly:

- Enable disappearing messages for all conversations
- Disable notification previews on the lock screen
- Register a phone number used exclusively for shelter work
- Enable screen security to prevent screenshots

For staff who need号码 anonymity, consider VoIP numbers from privacy-respecting providers rather than primary personal numbers.

### Email Configuration

When staff must communicate via email, enforce PGP encryption for sensitive threads:

```bash
# Generate a dedicated work key (Ed25519 for modern compatibility)
gpg --full-generate-key
# Key type: ECC
# Curve: Curve25519
# Expiration: 1 year (rotate regularly)
```

Store private keys on hardware tokens when possible. This prevents key extraction if devices are compromised.

## Application-Level Location Protection

### Photo Metadata Scrubbing

Photos shared within shelter systems must have EXIF data removed. Many programming languages provide libraries for this:

```python
# Python example using piexif
import piexif
from PIL import Image

def remove_exif(image_path, output_path):
    """Remove all EXIF data from an image."""
    img = Image.open(image_path)
    data = list(img.getdata())
    img_no_exif = Image.new(img.mode, img.size)
    img_no_exif.putdata(data)
    img_no_exif.save(output_path, "JPEG")
```

For batch processing, use ExifTool:

```bash
# Remove all metadata from images
exiftool -all= -overwrite_original *.jpg
```

### Coordinate Obfuscation

When location data must appear in systems, implement coordinate fuzzing:

```python
import random
import math

def fuzz_coordinates(lat, lon, radius_meters=500):
    """Add random offset to coordinates."""
    # Approximate: 0.01 degree ≈ 1.1km
    offset = radius_meters / 111000
    return (
        lat + random.uniform(-offset, offset),
        lon + random.uniform(-offset, offset)
    )
```

This allows general area display without exposing exact addresses.

## Operational Security Patterns

### Incident Response Communication

When responding to emergencies, use communication channels that don't log metadata extensively:

- Prefer Signal to SMS or standard voice calls
- Use burn-after-reading for temporary communications
- Establish predetermined code words for location references
- Have backup communication plans when primary channels fail

### Data Minimization in Records

Shelter databases should collect only essential information:

- Store addresses as general areas rather than exact coordinates
- Use case numbers instead of names where possible
- Implement data retention policies that delete old records automatically
- Encrypt sensitive fields at rest using envelope encryption

```sql
-- Example: Storing approximate location
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    case_number VARCHAR(20) UNIQUE,
    general_area VARCHAR(100),  -- Neighborhood/district only
    intake_date DATE,
    notes_encrypted BYTEA
);
```

## Physical Security Considerations

Technical measures fail without physical security discipline:

- Devices should require biometric or hardware-backed authentication
- Screens should lock automatically after short timeouts (1-2 minutes)
- Physical documents should be stored in locked cabinets
- Staff should avoid working with sensitive data in public spaces
- Shred all printed materials containing survivor information

## Conclusion

Protecting location data for domestic abuse shelter staff requires layered defenses. No single measure suffices, but combining device hardening, network protection, secure communications, and operational discipline creates meaningful protection. The implementations above provide starting points for developers building shelter systems or staff hardening their own setups.

Review and update these measures regularly. Adversaries adapt, and security requires ongoing attention.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
