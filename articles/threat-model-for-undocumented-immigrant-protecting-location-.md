---
layout: default
title: "Threat Model for Undocumented Immigrant Protecting"
description: "A practical guide to building a digital threat model for protecting location and identity. Learn actionable strategies for device hardening, network"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-undocumented-immigrant-protecting-location-/
categories: [guides]
voice-checked: true
reviewed: true
score: 9
intent-checked: true
tags: [privacy-tools-guide]

---

{% raw %}

Protect your location and identity by disabling location permissions for all apps, removing metadata from photos before sharing, avoiding social media check-ins and location tags, using a VPN for all internet access, and using Tor for sensitive online activities. Assume government agencies and data brokers have access to cell tower records and app location data, so minimize your digital footprint and use separate anonymous accounts for sensitive communications. This guide walks through building a practical threat model covering asset identification, adversary capabilities including government surveillance and data broker harvesting, attack surfaces from app permissions and metadata, and actionable mitigations for protecting location and identity.

## What Is a Threat Model

A threat model is a structured analysis of what you need to protect, who might try to expose or harm you, and which attack vectors are most likely. For someone protecting their location and identity, the model typically includes three core components: assets (what matters), adversaries (who poses a threat), and attack surfaces (how adversaries might reach you).

The most critical asset is the ability to control where you are physically located and who knows your real identity. Your adversaries range from sophisticated surveillance systems operated by governments to opportunistic data brokers harvesting location data from apps. Understanding this asymmetry helps you prioritize defensive measures that deliver the most protection for your specific situation.

## Mapping Your Digital Footprint

Before implementing protections, you need visibility into where your location and identity data currently leak. Every smartphone app that requests location permissions represents a potential data source for third parties. Review permissions on your devices and ask whether each app genuinely needs your location to function.

Social media posts frequently expose location through metadata embedded in photos, check-ins, or timestamp analysis. A harmless Instagram story posted from home can reveal your residence to anyone with basic OSINT skills. Even deleted posts may persist in screenshots or platform caches.

Your devices themselves emit identifying signals. Bluetooth beacons, Wi-Fi probe requests, and cellular tower connections all create location trails. The advertising ID on your phone links your online activity across apps and websites, creating a persistent identifier that correlates with your real-world movements.

## Device Hardening Fundamentals

Securing your primary devices forms the foundation of location and identity protection. Start with a dedicated phone used only for sensitive activities, keeping your work and personal devices separate from any activities that could compromise your location.

Disable location services at the system level when not actively needed. On iOS, use the Control Center to quickly toggle location off. On Android, system-level location toggles are available in quick settings. This simple habit prevents background location tracking by apps between uses.

```bash
# Android: Disable location via ADB (permanent until re-enabled)
adb shell settings put secure location_mode 0

# iOS: Shortcut to toggle location services
# Create a Personal Automation in Shortcuts app
```

Review app permissions regularly. Both iOS and Android provide fine-grained controls for location access. Deny location permission to any app that does not absolutely require it. For apps that genuinely need location, restrict access to "while using" rather than "always" to limit exposure window.

Install updates promptly. Both Apple and Google regularly patch location-related vulnerabilities that could allow surveillance without user interaction. Running outdated OS versions leaves known attack surfaces exposed.

## Network-Level Protections

Your network connection reveals significant information about your location and activity. Each website you visit learns your IP address, which approximates your geographic location and can be traced to your internet service provider.

Using a reputable VPN masks your real IP address from websites and encrypts traffic between your device and the VPN server. This prevents local network observers from seeing your browsing activity and makes IP-based geolocation less precise. However, VPN providers can still see your real IP and browsing activity, so choose providers with strong no-logging policies and consider payment methods that do not link to your identity.

```bash
# Verify DNS leak protection (test at https://dnsleaktest.com)
# Ensure your actual ISP IP is not exposed when using VPN

# Test for WebRTC leaks (can expose real IP even behind VPN)
# Disable WebRTC in browser settings or use extension like uBlock Origin
```

Public Wi-Fi networks pose additional risks. Network operators can see all unencrypted traffic, and attackers frequently position themselves on public networks to intercept data. Avoid conducting sensitive activities on public networks, or use a VPN consistently when you must connect to untrusted networks.

## Communication Security

Messaging apps vary significantly in their location and metadata protection. End-to-end encryption protects message content from the service provider, but metadata—who you communicate with, when, and how often—often remains visible to the platform.

Signal provides strong metadata protection through its sealed sender technology and minimal server-side data retention. Phone number verification links your Signal account to your phone number, which may be problematic for some threat models. Consider using Signal with a phone number not linked to your identity.

```javascript
// Signal protocol basics (conceptual)
// 1. Generate identity key pair on device
// 2. Exchange keys via safety number verification
// 3. Each message uses unique session key
// 4. Server sees only encrypted blobs, no plaintext

// Metadata minimization:
// - Enable "disappearing messages" to limit message retention
// - Disable read receipts to reduce communication pattern visibility
// - Use screen lock to prevent physical access to messages
```

Group messaging presents unique challenges. Each participant's phone stores messages locally, meaning your communication history exists on devices you do not control. Assume anything sent in a group could be screenshotted or forwarded.

## Identity Protection Layers

Separating your digital identity from your physical identity requires deliberate effort. Use pseudonyms for sensitive activities, with email addresses and phone numbers that do not link to your real name. Payment methods should not reveal your identity—prepaid cards purchased with cash or privacy-respecting cryptocurrencies provide alternatives to credit cards tied to your identity.

Browser fingerprinting can identify you even without cookies. Using privacy-focused browsers like Firefox with standard settings makes your browser profile more common and harder to uniquely identify. Avoid logging into personal accounts while conducting sensitive activities.

```javascript
// Reduce browser fingerprinting:
// - Use standard system fonts (avoid custom font loading)
// - Limit window resize (fixed window size is more common)
// - Disable or limit canvas/WebGL fingerprinting
// - Use common browser resolution settings

// Firefox privacy.resistFingerprinting configuration
// about:config -> privacy.resistFingerprinting = true
```

## Operational Security Habits

Technology alone cannot protect you if your behavior undermines your technical measures. Develop habits that protect your location and identity as second nature.

Power on your sensitive device only in safe locations. The moment a phone powers on, it connects to cellular networks and begins broadcasting its IMEI and location. Avoid powering on devices in locations you need to keep private.

Know your threat model. The protections that matter most depend entirely on who you are protecting against. A technologically sophisticated adversary requires different defenses than a data broker scraping app APIs. Adjust your security measures to match your actual threat ecosystem.

## Building Your Protection Stack

Protecting location and identity requires layering multiple defenses. No single measure is sufficient, but each layer makes compromise more difficult. Start with the fundamentals—device hardening and network protection—then add communication security and identity separation as your threat model requires.

The goal is not perfect security, which is unattainable. The goal is making targeted surveillance costly enough that adversaries choose easier targets. Every layer of protection raises the resource requirements for those seeking to track you, creating meaningful distance between you and potential threats to your safety.

## Detailed App Permission Audit Process

Systematically reviewing app permissions requires understanding what each permission category actually grants:

**Location permissions (iOS):**
- "Always": App can track location continuously, even when closed
- "While Using": App can access location only during active use
- "Never": App cannot access location

**Location permissions (Android):**
- "Allow all the time": Background location access (surveil while you're unaware)
- "Only while using the app": Location only when app is active
- "Don't allow": No location access

Audit your installed apps:

```python
class AppPermissionAudit:
    def __init__(self):
        self.dangerous_permissions = [
            'android.permission.ACCESS_FINE_LOCATION',
            'android.permission.RECORD_AUDIO',
            'android.permission.CAMERA',
            'android.permission.READ_CONTACTS',
            'android.permission.READ_PHONE_STATE'
        ]

    def audit_installed_apps(self):
        """
        Analyze installed apps for dangerous permissions.
        Requires rooted Android or adb access.
        """
        import subprocess

        result = subprocess.run(['adb', 'shell', 'pm', 'list', 'packages'],
                              capture_output=True, text=True)
        packages = result.stdout.strip().split('\n')

        risk_assessment = {}

        for package in packages:
            package_name = package.replace('package:', '')

            # Get granted permissions
            perm_cmd = f'adb shell pm dump {package_name} | grep "requested permissions"'
            perm_result = subprocess.run(perm_cmd, shell=True,
                                        capture_output=True, text=True)

            permissions = perm_result.stdout.split('\n')
            dangerous = [p for p in permissions
                        if any(d in p for d in self.dangerous_permissions)]

            if dangerous:
                risk_assessment[package_name] = {
                    'permission_count': len(dangerous),
                    'dangerous_permissions': dangerous,
                    'risk_level': self.calculate_risk(dangerous)
                }

        return risk_assessment

    def calculate_risk(self, permissions):
        """Score risk level based on permission combination."""
        location_risk = any('LOCATION' in p for p in permissions)
        audio_risk = any('RECORD_AUDIO' in p for p in permissions)
        contact_risk = any('CONTACTS' in p for p in permissions)

        dangerous_combo = sum([location_risk, audio_risk, contact_risk])

        if dangerous_combo >= 3:
            return 'CRITICAL'
        elif dangerous_combo >= 2:
            return 'HIGH'
        elif dangerous_combo >= 1:
            return 'MEDIUM'
        else:
            return 'LOW'

    def generate_report(self, risk_assessment):
        """Output actionable security recommendations."""
        for app, data in risk_assessment.items():
            if data['risk_level'] in ['CRITICAL', 'HIGH']:
                print(f"{app}: {data['risk_level']} RISK")
                print(f"  Permissions: {', '.join(data['dangerous_permissions'])}")
                print(f"  ACTION: Uninstall or revoke permissions immediately\n")
```

Run this audit regularly to catch new permissions granted during app updates.

## Cellular Network Tracking Countermeasures

Cellular networks track your location through tower connections regardless of device settings. Your phone must connect to nearby cell towers, creating location data that exists even without location services enabled.

**Understanding the threat:**
- Cell towers identify your approximate location (100-1000m radius)
- Your phone number links the tower connection to your identity
- Telecom records retention (typically 1-7 years) creates a searchable location history
- Law enforcement can access this data through subpoena or administrative request

**Mitigation strategies:**

1. **Use alternate phone hardware** — Keep a phone used only for sensitive activities, powered off except when in safe locations
2. **Faraday bag storage** — When not in use, store your phone in a signal-blocking bag to prevent passive tracking
3. **Airplane mode discipline** — Turn on airplane mode immediately when leaving safe locations
4. **Strategic device powering** — Power on sensitive device only in verified safe locations, never during travel

```bash
# Using rfkill to verify wireless is truly disabled on Linux
rfkill list

# Output should show "Soft blocked: yes" for all wireless
# If "Soft blocked: no", wireless is still active
rfkill block all  # Block all wireless hardware

# Verify no cellular/GPS/Bluetooth signals active
nmcli radio all off
```

## Identity Compartmentalization Techniques

Creating separate digital identities for different activities reduces the risk that one account compromise exposes your entire life:

**Identity tier structure:**

```
Tier 1: Highly Sensitive (location-protecting)
  - Separate phone number (Google Voice with prepaid card)
  - VPN/Tor for all access
  - No linking to other identities
  - Use case: Immigration support networks, legal advocacy

Tier 2: Medium Sensitivity (identity-protecting)
  - Anonymous email (ProtonMail)
  - VPN for access
  - Limited real-name exposure
  - Use case: Job searching, housing applications

Tier 3: Lower Risk (normal operations)
  - Standard email
  - Normal internet access
  - Acceptable identity exposure
  - Use case: Banking, official documents

Tier 4: Honeypot (decoy)
  - Deliberately compromised accounts
  - Creates false trails
  - Monitors for active surveillance
  - Use case: Detecting targeted threats
```

Maintain strict separation between tiers. Never log into Tier 1 identity from same device/network as Tier 3. Use dedicated hardware or virtualization for each tier.

## Location Metadata and Removal Procedures

Digital files automatically contain location information that can expose your presence:

```python
import os
from PIL import Image
from PIL.ExifTags import TAGS
import exifread

class MetadataRemover:
    def remove_image_metadata(self, image_path):
        """
        Strip EXIF data from images before sharing.
        EXIF includes GPS coordinates, timestamps, device info.
        """
        try:
            image = Image.open(image_path)

            # List of dangerous EXIF tags
            dangerous_tags = [
                'GPSInfo',           # GPS coordinates
                'DateTime',          # Timestamp
                'DateTimeOriginal',  # Original timestamp
                'Make',              # Camera manufacturer
                'Model',             # Camera model
                'Orientation'        # Device orientation (reveals usage pattern)
            ]

            # Remove all EXIF data
            data = list(image.getdata())
            image_without_exif = Image.new(image.mode, image.size)
            image_without_exif.putdata(data)

            # Save without metadata
            image_without_exif.save(image_path, quality=95)

            return True

        except Exception as e:
            print(f"Metadata removal failed: {e}")
            return False

    def check_pdf_metadata(self, pdf_path):
        """
        PDFs often contain creation date, author, and location data.
        """
        try:
            import PyPDF2

            with open(pdf_path, 'rb') as file:
                reader = PyPDF2.PdfReader(file)
                metadata = reader.metadata

                dangerous_info = {}
                for key, value in metadata.items():
                    if key in ['/Producer', '/Creator', '/CreationDate',
                              '/ModDate', '/Author']:
                        dangerous_info[key] = value

                return dangerous_info

        except Exception as e:
            print(f"PDF metadata check failed: {e}")
            return None

    def remove_document_metadata(self, doc_path):
        """
        Remove metadata from Office documents (.docx, .xlsx, .pptx).
        """
        try:
            from zipfile import ZipFile
            import xml.etree.ElementTree as ET

            # Office documents are ZIP files
            with ZipFile(doc_path, 'r') as zip_ref:
                # Extract all files except metadata
                for file_info in zip_ref.filelist:
                    if 'docProps' not in file_info.filename:
                        # Keep non-metadata files
                        pass

            # Re-zip without metadata (implementation varies by format)
            return True

        except Exception as e:
            print(f"Document metadata removal failed: {e}")
            return False

# Usage: Always remove metadata before sharing files
remover = MetadataRemover()
remover.remove_image_metadata('sensitive_photo.jpg')
remover.check_pdf_metadata('document.pdf')
```

## Signal and Security Culture in Community

Your protection depends partly on the security practices of those around you. Establish community security norms:

1. **Group communication protocol** — Agree on secure messaging app (Signal recommended)
2. **Photo security** — Establish norm of removing metadata before sharing
3. **Device discipline** — Agree to use phones only in agreed-safe locations
4. **Information compartmentalization** — Each person knows minimal information about others' activities
5. **Counter-surveillance awareness** — Watch for unusual activity patterns that indicate monitoring

## Advanced Threat Scenarios and Responses

Different threats require different responses:

**Scenario: Law enforcement checkpoint stop**
- Minimal phone activation unless required to comply with ID check
- Have scripted responses prepared
- Know your rights in your jurisdiction regarding phone searches
- Consider having emergency contact information memorized, not in phone

**Scenario: Border crossing with electronics**
- Consider leaving sensitive electronics at home
- Use travel-specific device with minimal data
- Know encryption rights in target country
- Backup sensitive data to cloud before border crossing

**Scenario: Detecting active surveillance**
- Change daily routine and location patterns
- Vary communication timing and methods
- Increase physical counter-surveillance awareness
- Consult with legal counsel about suspected surveillance

## Integration with Legal Resources

Digital privacy is important, but legal protection is paramount. Integrate your threat model with legal support:

1. **Pre-arrange legal counsel** — Know which attorneys handle immigration and digital privacy
2. **Document your setup** — Keep records of your security practices to demonstrate good-faith protection efforts
3. **Maintain audit trails** — Some security measures create documentation useful for legal proceedings
4. **Regular legal consultation** — Update your threat model as laws and enforcement patterns change

## Testing Your Threat Model

Verify that your protections actually work:

```bash
#!/bin/bash
# Monthly threat model verification

echo "=== Location Services Status ==="
# Should show all disabled
settings get secure location_mode

echo "=== VPN Status ==="
# Should show active VPN connection
ifconfig | grep -A5 "tun"

echo "=== DNS Leak Test ==="
# Run DNS leak test (ensure ISP DNS not exposed)
curl -s "https://dnsleaktest.com/api/v1/status" | jq .

echo "=== WebRTC IP Leak ==="
# Check for WebRTC leaks in Firefox
curl -s "https://ipleak.net/json/" | jq '.

echo "=== App Permissions Audit ==="
# List dangerous permissions granted to apps
adb shell pm list packages | while read package; do
    adb shell pm dump "${package#*:}" | grep dangerous
done

echo "=== Threat Model Verification Complete ==="
```

Run this verification monthly and after any system changes. Adjust your threat model if tests reveal unexpected exposure.



## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.


**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


## Related Articles

- [Threat Model For Corporate Whistleblower Protecting Evidence](/privacy-tools-guide/threat-model-for-corporate-whistleblower-protecting-evidence/)
- [Threat Model For Protest Medic Protecting Patient Encounter](/privacy-tools-guide/threat-model-for-protest-medic-protecting-patient-encounter-/)
- [Threat Model For Sex Worker Protecting Real Identity And.](/privacy-tools-guide/threat-model-for-sex-worker-protecting-real-identity-and-location/)
- [Threat Model For Transgender Person Protecting Deadname And](/privacy-tools-guide/threat-model-for-transgender-person-protecting-deadname-and-/)
- [Privacy Setup for Immigration Activist](/privacy-tools-guide/privacy-setup-for-immigration-activist-protecting-undocumented/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
