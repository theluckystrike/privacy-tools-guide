---
layout: default
title: "Threat Model For Union Organizer In Hostile Employer"
description: "A technical guide for union organizers to build threat models when operating in hostile employer environments. Includes practical"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /threat-model-for-union-organizer-in-hostile-employer-environ/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Union organizers in hostile employer environments need a threat model that accounts for corporate surveillance capabilities—email monitoring, badge swipes, video surveillance, and potential infiltration—while remaining practical for organizing work. This guide uses the STRIDE framework adapted for union contexts, showing you how to identify your assets (communication metadata, membership lists, meeting locations), map adversary capabilities, and implement concrete countermeasures that developers and power users can deploy immediately.

## Understanding Your Adversary

Hostile employers often possess significant resources: corporate security teams, legal departments, private investigators, and sometimes access to sophisticated surveillance technology. Your threat model must account for these capabilities while remaining practical for organizing efforts.

Start by mapping what an adversary can theoretically do versus what they actually do. Corporate security teams typically have access to:
- Badge swipes and building access logs
- Email monitoring on company devices
- Video surveillance in workplaces
- Social media monitoring tools
- Possibility of "labor spy" infiltration

## Building Your Threat Model

A practical threat model uses the STRIDE framework adapted for organizing contexts: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, and Elevation of Privilege.

### Step 1: Asset Identification

Identify what you need to protect. For union organizers, this typically includes:

- Communication metadata (who contacted whom, when)
- Membership lists and contact information
- Meeting locations and schedules
- Internal strategy discussions
- Personal devices and accounts

Create a simple asset inventory in a structured format:

```python
# assets.py - Simple asset inventory for threat modeling
assets = {
    "communications": {
        "sensitivity": "high",
        "examples": ["Signal messages", "emails", "meeting notes"],
        "adversary_interest": "who is involved, leadership structure"
    },
    "member_data": {
        "sensitivity": "critical",
        "examples": ["names", "phone numbers", "work locations"],
        "adversary_interest": "identify organizers, map social networks"
    },
    "strategy_documents": {
        "sensitivity": "high",
        "examples": ["organizing plans", "talking points", "timelines"],
        "adversary_interest": "anticipate actions, identify weaknesses"
    }
}
```

### Step 2: Threat Actor Analysis

Document specific threats using a severity matrix:

| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|-------------|
| Device confiscation | Medium | Critical | Full disk encryption, minimal local storage |
| Communication interception | High | High | End-to-end encryption, ephemeral messages |
| Social engineering | High | Medium | Training, verification protocols |
| Physical surveillance | Medium | Medium | OPSEC procedures, counter-surveillance |

### Step 3: Attack Vector Mapping

Consider how each adversary might access your assets:

**Digital attack vectors:**
- Phishing emails targeting organizing email addresses
- Compromised accounts through password reuse
- Malware on personal devices
- SIM swapping for phone number takeover
- Keyloggers on shared or public computers

**Physical attack vectors:**
- Shoulder surfing in public spaces
- Document theft from cars or homes
- Planted informants at meetings
- Trash diving for discarded materials

## Practical Countermeasures

### Secure Communication Architecture

Use a defense-in-depth approach to communications:

```bash
# Signal configuration for high-security use
# Enable disappearing messages by default
signal-cli -u +1234567890 set-expiration --expiration 3600 *

# Use Signal groups with strict membership
# Create separate groups for different organizing activities
# Never use work devices for union communications
```

For sensitive discussions, consider layered encryption:

```python
# Use multiple encryption layers for sensitive communications
# Layer 1: Signal (transport encryption)
# Layer 2: PGP for additional protection on stored messages

from cryptography.fernet import Fernet
import base64

def double_encrypt(message, key1, key2):
    """Apply two layers of Fernet encryption"""
    f1 = Fernet(key1)
    f2 = Fernet(key2)
    # First layer
    encrypted = f1.encrypt(message.encode())
    # Second layer
    double_encrypted = f2.encrypt(encrypted)
    return base64.b64encode(double_encrypted)
```

### Device Security Checklist

Implement these technical controls on devices used for organizing:

1. **Enable full disk encryption** - FileVault on macOS, BitLocker on Windows, LUKS on Linux
2. **Use a strong passphrase** - Minimum 20 characters, stored in password manager
3. **Enable automatic updates** - Security patches within 48 hours of release
4. **Use a separate device** - Dedicated phone and laptop for organizing work
5. **Configure secure messaging** - Signal with disappearing messages enabled
6. **Enable Find My Device** - Remote wipe capability for lost/stolen devices
7. **Use VPN always** - Protect network traffic from local surveillance

### Operational Security Procedures

Establish routines that minimize risk exposure:

**Meeting security:**
- Rotate meeting locations frequently
- Use code words for sensitive discussions
- Designate security observers
- Search for surveillance devices before sensitive meetings
- Leave devices outside meeting rooms

**Information handling:**
- Need-to-know distribution for member data
- Regular purging of unnecessary communications
- Secure deletion of strategy documents after campaigns
- Physical security for paper documents

## Incident Response Planning

Prepare for potential security incidents with a documented response plan:

```yaml
# incident-response.yaml
incident_response:
  device_compromised:
    - Immediately disconnect from networks
    - Do not power off (preserves RAM for forensics)
    - Contact technical security support
    - Document everything

  account_compromised:
    - Change passwords from clean device
    - Enable two-factor authentication
    - Review recent account activity
    - Notify affected contacts

  suspicious_surveillance:
    - Note times, locations, descriptions
    - Photograph vehicles if safe
    - Report to legal counsel
    - Adjust security protocols
```

## Continuous Assessment

Threat models require regular updates as conditions change. Schedule quarterly reviews of:

- New adversary capabilities or resources
- Changes in organizing strategy that alter risk profile
- Technology updates that affect security posture
- Lessons learned from near-misses or incidents

Document these reviews and adjust countermeasures accordingly. Security is an ongoing process, not an one-time configuration.



## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.


**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


## Related Articles

- [Threat Model Assessment For High Risk Journalist In Hostile](/privacy-tools-guide/threat-model-assessment-for-high-risk-journalist-in-hostile-/)
- [Privacy Tools For Union Organizer Protecting Member Communic](/privacy-tools-guide/privacy-tools-for-union-organizer-protecting-member-communic/)
- [Threat Model For Activist In Authoritarian Country Digital S](/privacy-tools-guide/threat-model-for-activist-in-authoritarian-country-digital-s/)
- [Threat Model For Corporate Whistleblower Protecting Evidence](/privacy-tools-guide/threat-model-for-corporate-whistleblower-protecting-evidence/)
- [Threat Model For Human Rights Worker In Conflict Zone Guide](/privacy-tools-guide/threat-model-for-human-rights-worker-in-conflict-zone-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
