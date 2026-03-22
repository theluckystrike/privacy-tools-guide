---
layout: default
title: "Threat Model For Protest Medic Protecting Patient Encounter"
description: "A technical guide for developers and power users on building a threat model to protect patient encounter information as a protest medic"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-protest-medic-protecting-patient-encounter-/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Protect protest medic patient data by documenting encounters without storing identifiers (encrypted case numbers only), using encrypted messaging (Signal) for coordination, never storing names/locations together, and destroying paper notes securely. Anticipate law enforcement subpoenas and device seizure by keeping encrypted offline storage separate from operational devices.

## Table of Contents

- [Who Wants Your Data](#who-wants-your-data)
- [Digital Footprint Assessment](#digital-footprint-assessment)
- [Secure Encounter Documentation](#secure-encounter-documentation)
- [Device Security Fundamentals](#device-security-fundamentals)
- [Communication Security](#communication-security)
- [Data Retention Strategy](#data-retention-strategy)
- [Physical Security Considerations](#physical-security-considerations)
- [Incident Response Plan](#incident-response-plan)
- [Implementation Checklist](#implementation-checklist)

## Who Wants Your Data

Before implementing protections, identify the adversaries. Protest medic data faces several threat actors:

- **Law enforcement**: Subpoenas for medical records, real-time surveillance of communications, device seizure at protests
- **Event organizers**: May request incident reports for liability purposes
- **Data brokers**: Aggregating health-related information from multiple sources
- **Bad actors**: Individuals who may target medics for harassment or doxxing

The core principle: assume any data you create could eventually be accessed by an adversary. Design your systems accordingly.

## Digital Footprint Assessment

Start by mapping where patient information flows through your systems:

```bash
# Audit your current data storage
echo "=== Current Documentation Methods ==="
echo "1. Paper notebooks - where stored?"
echo "2. Phone notes - which apps?"
echo "3. Signal/Telegram messages - group chats?"
echo "4. Cloud storage - which providers?"
echo "5. Email - personal or work accounts?"
```

Understanding your current data flow reveals vulnerabilities. Most medics inadvertently store patient information across multiple platforms, creating a fragmented attack surface.

## Secure Encounter Documentation

The most practical solution for protest medics is documenting encounters without storing identifying information. Here's a technical approach:

```python
#!/usr/bin/env python3
"""
Protest Medic Encounter Logger - Minimal Data Version
Stores only anonymized treatment codes, not patient identities
"""
import json
import hashlib
from datetime import datetime
from pathlib import Path

class SecureEncounterLog:
    def __init__(self, encryption_key=None):
        self.encounters = []
        self.encryption_key = encryption_key  # Optional: AES key for storage encryption

    def log_encounter(self, treatment_codes, notes="", location="", timestamp=None):
        """
        Log encounter using treatment codes instead of patient identifiers

        treatment_codes: List of standardized codes (e.g., ["BLK", "TEAR", "HYP"])
        - BLK: Basic life support
        - TEAR: Tear gas exposure treatment
        - HYP: Heat-related illness
        - TRA: Trauma/wound care
        - MHA: Mental health acute crisis
        """
        if timestamp is None:
            timestamp = datetime.now().isoformat()

        # Create anonymized encounter record
        encounter = {
            "timestamp": timestamp,
            "location_code": self._anonymize_location(location),
            "treatments": treatment_codes,
            # Notes should NEVER contain:
            # - Names, physical descriptions that could identify individuals
            # - Specific conversation content
            # - Photos that could identify people
            "notes": notes,
            "incident_id": self._generate_incident_id(timestamp, location)
        }

        self.encounters.append(encounter)
        return encounter["incident_id"]

    def _anonymize_location(self, location):
        """Convert specific location to area code only"""
        if not location:
            return "UNK"
        # Return only neighborhood/area, not specific address
        return hashlib.sha256(location.encode()).hexdigest()[:8].upper()

    def _generate_incident_id(self, timestamp, location):
        """Generate non-sequential incident ID"""
        data = f"{timestamp}{location}".encode()
        return hashlib.sha256(data).hexdigest()[:12].upper()

    def export_encounters(self, filepath):
        """Export to encrypted JSON"""
        data = json.dumps(self.encounters, indent=2)

        if self.encryption_key:
            # Use cryptography library for actual encryption
            from cryptography.fernet import Fernet
            f = Fernet(self.encryption_key)
            encrypted = f.encrypt(data.encode())
            with open(filepath, 'wb') as f:
                f.write(encrypted)
        else:
            with open(filepath, 'w') as f:
                f.write(data)

# Usage example
logger = SecureEncounterLog()
incident_id = logger.log_encounter(
    treatment_codes=["TEAR", "BLK"],
    notes="Multiple patients at intersection. Tear gas exposure, eye irrigation performed.",
    location="Downtown Plaza"
)
print(f"Logged incident: {incident_id}")
```

This approach records what treatments were provided without linking them to specific individuals. The location gets hashed to a code rather than storing exact addresses.

## Device Security Fundamentals

Your phone and any devices containing encounter notes become high-value targets. Implement these baseline protections:

```bash
# iOS: Enable these settings immediately
# 1. Settings > Face ID/Touch ID & Passcode - Strong passcode (6+ digits)
# 2. Settings > Face ID/Touch ID & Passcode - Erase Data after 10 failed attempts
# 3. Settings > Privacy - Disable Analytics sharing
# 4. Settings > Privacy > Location Services - Set to "While Using" only for essential apps

# Android (AOSP):
# 1. Settings > Security - Strong lock (PIN/biometric)
# 2. Settings > Security - Encrypt phone (full disk encryption)
# 3. Settings > Apps > Permissions - Audit all permissions
# 4. Settings > Network > Advanced - Disable WiFi scanning

# Both platforms:
# - Enable encrypted messaging (Signal)
# - Use separate work phone for protest activities if possible
# - Disable cloud backup for sensitive apps
```

## Communication Security

When coordinating with other medics or communicating about encounters, use these protocols:

```python
# Signal group security checklist:
"""
1. Create NEW group for each event (don't reuse)
2. Enable disappearing messages (24-hour default)
3. Verify all member identities in person
4. Set group avatar to generic symbol (no identifying photos)
5. Add ONLY members who need to know
6. Never discuss specific patients by any identifying detail
7. Use code words:
   - "Patient" → "P" or "individual"
   - "Tear gas" → "gas" or "chemical"
   - "Arrested" → "taken" or "removed"
"""

# For group chats, establish protocol:
GROUP_PROTOCOL = """
Signal Security Protocol:
- Disappearing messages: 24 hours
- No names in messages - use incident codes
- Delete chat history after each event
- Verify group members in person at start of shift
- If device seized, no message content visible (Signal doesn't store)
"""
```

## Data Retention Strategy

Implement strict retention policies. The safest data is data you never keep:

```bash
# Automated cleanup script for encounter notes
#!/bin/bash
# Run daily via cron - deletes notes older than 30 days

NOTES_DIR="$HOME/Documents/protest-notes"
DAYS_TO_KEEP=30

find "$NOTES_DIR" -type f -mtime +$DAYS_TO_KEEP -delete
echo "Cleaned notes older than $DAYS_TO_KEEP days"
```

For paper notes:
- Use tear-resistant paper (Rite in the Rain)
- Write in a notation system only you understand
- Store in secure location (not at home if warrants possible)
- Destroy after 30 days or after legal risk passes

## Physical Security Considerations

Digital protections fail without physical security:

- **Device seizure**: Use device encryption, don't unlock with biometrics (can be compelled), use strong passcodes
- **Home raids**: Don't store patient information at home if possible, use secure cloud with legal protection
- **Protests**: Leave unnecessary devices at home, use airplane mode, consider Faraday bag

## Incident Response Plan

If your device is seized or you're detained:

```
1. DON'T unlock devices with face/fingerprint (legally compellable)
2. Provide passcode only after legal consultation
3. State: "I invoke my right to remain silent"
4. Don't discuss patient information - it's protected
5. Contact attorney before explaining anything
6. Document everything you remember about the interaction
```

## Implementation Checklist

Use this checklist when setting up your protest medic security:

- [ ] Create dedicated email for medic communications
- [ ] Set up Signal with disappearing messages
- [ ] Enable full-disk encryption on all devices
- [ ] Implement code words with team
- [ ] Set up automated note cleanup
- [ ] Create anonymized encounter logging system
- [ ] Establish data retention policy
- [ ] Train team on incident response
- [ ] Remove identifying photos from devices before events
- [ ] Use separate phone if possible

The goal isn't paranoia—it's reasonable protection for the people who trust you with their care. By implementing these measures, you reduce the risk that patient information becomes evidence, gets leaked, or harms the people you treated.

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

- [Threat Model For Medical Marijuana Patient In Non Legal Stat](/privacy-tools-guide/threat-model-for-medical-marijuana-patient-in-non-legal-stat/)
- [Threat Model For Corporate Whistleblower Protecting Evidence](/privacy-tools-guide/threat-model-for-corporate-whistleblower-protecting-evidence/)
- [Threat Model For Sex Worker Protecting Real Identity And.](/privacy-tools-guide/threat-model-for-sex-worker-protecting-real-identity-and-location/)
- [Threat Model For Transgender Person Protecting Deadname And](/privacy-tools-guide/threat-model-for-transgender-person-protecting-deadname-and-/)
- [Threat Model for Undocumented Immigrant Protecting.](/privacy-tools-guide/threat-model-for-undocumented-immigrant-protecting-location-/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
