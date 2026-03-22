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
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Union organizers in hostile employer environments need a threat model that accounts for corporate surveillance capabilities—email monitoring, badge swipes, video surveillance, and potential infiltration—while remaining practical for organizing work. This guide uses the STRIDE framework adapted for union contexts, showing you how to identify your assets (communication metadata, membership lists, meeting locations), map adversary capabilities, and implement concrete countermeasures that developers and power users can deploy immediately.

## Table of Contents

- [Understanding Your Adversary](#understanding-your-adversary)
- [Building Your Threat Model](#building-your-threat-model)
- [Practical Countermeasures](#practical-countermeasures)
- [Incident Response Planning](#incident-response-planning)
- [Continuous Assessment](#continuous-assessment)
- [Advanced Counter-Surveillance Tradecraft](#advanced-counter-surveillance-tradecraft)
- [Legal Defense Framework](#legal-defense-framework)
- [Secure Information Sharing Infrastructure](#secure-information-sharing-infrastructure)
- [Risk Assessment Matrix](#risk-assessment-matrix)
- [Budgeting for Security](#budgeting-for-security)
- [Incident Response During Active Threat](#incident-response-during-active-threat)

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

Document these reviews and adjust countermeasures accordingly. Security is an ongoing process, not a one-time configuration.

## Advanced Counter-Surveillance Tradecraft

### TEMPEST and Eavesdropping Mitigation

Protect against sophisticated electronic surveillance:

```python
# Counter-surveillance device detection
class CounterSurveillanceAssessment:
    def __init__(self, meeting_location):
        self.location = meeting_location
        self.issues = []

    def check_for_hidden_devices(self):
        """Provide framework for checking for devices"""
        return {
            'visual_inspection': [
                'Check for unusual fixtures',
                'Look for small holes or lenses',
                'Examine smoke detectors, clocks, plants'
            ],
            'radio_frequency_check': [
                'Use RF detector in range 100MHz - 2.7GHz',
                'Check for active transmitters',
                'Scan multiple times from different positions'
            ],
            'physical_security': [
                'Verify windows are closed',
                'Confirm doors can be monitored',
                'Check for sight lines from outside'
            ]
        }

    def assess_acoustic_environment(self):
        """Assess for audio surveillance"""
        return {
            'ambient_noise': 'Is background noise sufficient? (60-70dB preferred)',
            'reflection_points': 'Avoid reflective surfaces opposite speakers',
            'background_music': 'Consider using white noise or music',
            'distance': 'Maintain 2-3 feet minimum distance for conversation'
        }

    def thermal_considerations(self):
        """Reduce thermal signatures"""
        return {
            'door_seals': 'Use weather stripping under doors',
            'window_coverage': 'Use thermal curtains',
            'room_temperature': 'Keep consistent with outside',
            'body_heat': 'Multiple people reduce individual heat signature'
        }
```

### Social Engineering Defense

Build resistance to infiltration and social engineering:

```yaml
Social Engineering Prevention:
  Training:
    - Recognize common pretexting techniques
    - Verify identity through established channels
    - Be suspicious of new members seeking information
    - Report unusual questions to security officer

  Operational Security:
    - Compartmentalize information (need-to-know)
    - Rotate meeting locations frequently
    - Use code words for sensitive discussions
    - Verify communications before acting

  Vetting Process:
    - Longer relationship building before trust
    - References from established members
    - Multiple interviews by different people
    - Observation of behavior patterns

  Red Flags:
    - Excessive interest in leadership structure
    - Questions about meeting schedules
    - Attempts to gain physical access
    - Suspicious documentation requests
```

## Legal Defense Framework

Prepare for potential legal actions:

```python
class LegalDefenseFramework:
    def __init__(self):
        self.evidence_collection = []
        self.document_retention = {}
        self.attorney_contact = {}

    def establish_attorney_relationship(self):
        """Establish relationship with labor attorney before problems occur"""
        return {
            'attorney_selection': [
                'Labor law specialty',
                'Experience with union defense',
                'Local jurisdiction expertise',
                'Confidentiality agreement'
            ],
            'retainer': 'Establish retainer before issues arise',
            'communication_protocol': 'Secure communication methods',
            'representation_scope': 'Define what attorney covers'
        }

    def document_adverse_actions(self):
        """Document all employer adverse actions"""
        adverse_event = {
            'date': '2026-03-15',
            'time': '14:30',
            'location': 'conference room A',
            'witnesses': ['name1', 'name2'],
            'description': 'detailed description',
            'why_adverse': 'explain connection to protected activity',
            'potential_damages': 'estimate impact',
            'evidence': [
                'emails',
                'messages',
                'performance reviews',
                'witnesses'
            ]
        }
        return adverse_event

    def preserve_electronic_evidence(self):
        """Properly preserve electronic evidence"""
        return {
            'email_archiving': [
                'Export emails before they are deleted',
                'Save in native format (PST/MBOX)',
                'Document chain of custody',
                'Use forensic tools if necessary'
            ],
            'message_preservation': [
                'Screenshot with metadata',
                'Export with timestamps',
                'Include full conversation threads',
                'Save before account deletion'
            ],
            'metadata_preservation': [
                'Keep file modification dates',
                'Preserve creation timestamps',
                'Document access logs',
                'Archive complete headers'
            ]
        }

    def retaliation_documentation_template(self):
        """Template for documenting retaliation"""
        return {
            'protected_activity': 'What union activity did you engage in?',
            'knowledge_of_activity': 'How did employer learn of it?',
            'timeline_of_actions': 'When did adverse action occur?',
            'causation': 'Why do you believe it was retaliation?',
            'witnesses': 'Who can corroborate the connection?',
            'damages': 'What harm resulted?',
            'notice_to_employer': 'When did you notify HR/employer?'
        }
```

## Secure Information Sharing Infrastructure

Build secure systems for organizing communication:

```bash
#!/bin/bash
# Setup secure information sharing infrastructure

setup_secure_communication() {
  # 1. Encrypted file sharing server
  # Requirements: Self-hosted, encrypted at rest, TLS in transit

  # Example using Nextcloud with encryption
  docker run -d \
    --name nextcloud \
    -e MYSQL_ROOT_PASSWORD="$SECURE_PASSWORD" \
    -e MYSQL_DATABASE="nextcloud" \
    -v /encrypted_volume/nextcloud:/var/www/html \
    nextcloud:latest

  # 2. Organize-specific chat/messaging
  # Use self-hosted Mattermost or Rocket.Chat

  docker run -d \
    --name mattermost \
    -e MM_SQLSETTINGS_DATASOURCENAME="$DB_CONNECTION" \
    -v /encrypted_volume/mattermost:/mattermost/data \
    mattermost/mattermost-team-edition:latest

  # 3. Encrypted document management
  # Setup with access controls

  setup_document_permissions() {
    # Only share documents with explicit recipients
    # Maintain audit logs of who accessed what
    # Implement expiring access tokens
    # Log all document downloads
  }
}
```

### Access Control Matrix

Define precise access controls:

```python
class SecureAccessControl:
    def __init__(self):
        self.roles = {
            'leadership': {
                'strategy_documents': True,
                'member_list': True,
                'meeting_locations': True,
                'legal_information': True
            },
            'member': {
                'strategy_documents': False,
                'member_list': False,
                'meeting_locations': True,
                'legal_information': False
            },
            'coordinator': {
                'strategy_documents': True,
                'member_list': False,
                'meeting_locations': True,
                'legal_information': False
            }
        }

    def grant_access(self, user_id, role):
        """Grant role with associated permissions"""
        if role not in self.roles:
            raise ValueError(f"Unknown role: {role}")

        # Update user role in system
        # Log access grant with reason and duration

    def audit_access(self):
        """Generate access audit log"""
        return {
            'timestamp': datetime.utcnow(),
            'access_grants': [],
            'access_revocations': [],
            'anomalies': []
        }
```

## Risk Assessment Matrix

Formalize threat evaluation:

```python
class ThreatRiskMatrix:
    def evaluate_threat(self, threat_name, likelihood, impact):
        """
        Likelihood: 1-5 (1=rare, 5=almost certain)
        Impact: 1-5 (1=minimal, 5=catastrophic)
        """
        risk_score = likelihood * impact
        risk_level = self._categorize_risk(risk_score)

        return {
            'threat': threat_name,
            'likelihood': likelihood,
            'impact': impact,
            'risk_score': risk_score,
            'risk_level': risk_level,
            'mitigation_required': risk_level in ['HIGH', 'CRITICAL']
        }

    def _categorize_risk(self, score):
        if score <= 3:
            return 'LOW'
        elif score <= 9:
            return 'MEDIUM'
        elif score <= 16:
            return 'HIGH'
        else:
            return 'CRITICAL'

    def prioritize_mitigations(self, threats):
        """Sort threats by risk level for resource allocation"""
        sorted_threats = sorted(
            threats,
            key=lambda x: x['risk_score'],
            reverse=True
        )

        return {
            'critical': [t for t in sorted_threats if t['risk_score'] >= 16],
            'high': [t for t in sorted_threats if 9 <= t['risk_score'] < 16],
            'medium': [t for t in sorted_threats if 3 < t['risk_score'] < 9],
            'low': [t for t in sorted_threats if t['risk_score'] <= 3]
        }
```

## Budgeting for Security

Allocate resources appropriately:

```python
security_budget_allocation = {
    'technology': {
        'encrypted_messaging': '$100/month',
        'vpn_services': '$60/month',
        'device_security': '$500 one-time',
        'password_management': '$50/month'
    },
    'training': {
        'initial_security_training': '$500',
        'quarterly_refresher': '$200',
        'new_member_onboarding': '$100'
    },
    'professional_services': {
        'security_audit': '$2000 one-time',
        'legal_consultation': '$300/month',
        'threat_modeling': '$1500 one-time'
    },
    'operations': {
        'counter_surveillance_equipment': '$1000',
        'safe_meeting_spaces': '$200/month',
        'document_destruction': '$50/month'
    }
}
```

## Incident Response During Active Threat

If security incident occurs:

```yaml
Incident Response Protocol:
  Detection (immediately):
    - Notify security officer
    - Preserve all evidence
    - Isolate affected systems
    - Contact attorney

  Containment (first hour):
    - Change critical passwords
    - Notify affected members
    - Pause organizing activities if necessary
    - Document timeline

  Investigation (first 24 hours):
    - Forensic analysis of affected systems
    - Interview witnesses
    - Gather evidence for legal case
    - Assess scope of breach

  Recovery (ongoing):
    - Restore clean systems
    - Revise security procedures
    - Conduct security training
    - Follow up with attorney

  Lessons Learned (1 week):
    - Update threat model
    - Modify procedures
    - Improve controls
    - Document for future reference
```

This threat model for union organizers emphasizes that security is both technical and procedural—both aspects are equally important for protecting your organizing work.

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
- [Threat Model For Political Dissident In Surveillance State](/privacy-tools-guide/threat-model-for-political-dissident-in-surveillance-state-2/)
- [Threat Model for Undocumented Immigrant Protecting](/privacy-tools-guide/threat-model-for-undocumented-immigrant-protecting-location-/)
- [Threat Model For Sex Worker Protecting Real Identity](/privacy-tools-guide/threat-model-for-sex-worker-protecting-real-identity-and-location/)
- [Threat Model For Transgender Person Protecting Deadname](/privacy-tools-guide/threat-model-for-transgender-person-protecting-deadname-and-/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
