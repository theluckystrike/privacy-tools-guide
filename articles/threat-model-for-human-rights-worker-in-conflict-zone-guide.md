---
---
layout: default
title: "Threat Model For Human Rights Worker In Conflict Zone Guide"
description: "A threat modeling guide for human rights workers in conflict zones. Learn to identify assets, analyze adversaries, and implement"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /threat-model-for-human-rights-worker-in-conflict-zone-guide/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Human rights workers in conflict zones face nation-state and paramilitary threats with capabilities including physical surveillance, device seizure with forensic extraction, social network analysis, and sophisticated network eavesdropping. Protect evidence of human rights abuses using Tails Linux for documentation work, Signal encrypted communications, encrypted storage vaults for sensitive documents, and regular device wipes. Establish secure communication with international organizations, separate your work device from personal phone, use dead drops or air-gapped document transfer, and implement strict access controls on victim testimony. This guide provides a practical threat modeling framework covering asset identification, adversary capability analysis, and actionable mitigations tailored to conflict zone environments.

## Table of Contents

- [Understanding the Threat Environment](#understanding-the-threat-environment)
- [Step 1: Identifying Critical Assets](#step-1-identifying-critical-assets)
- [Step 2: Identifying and Profiling Adversaries](#step-2-identifying-and-profiling-adversaries)
- [Step 3: Analyzing Attack Vectors](#step-3-analyzing-attack-vectors)
- [Step 4: Implementing Practical Mitigations](#step-4-implementing-practical-mitigations)
- [Step 5: Operational Security Procedures](#step-5-operational-security-procedures)
- [Continuous Threat Model Maintenance](#continuous-threat-model-maintenance)
- [Building a Tiered Security Approach](#building-a-tiered-security-approach)
- [Technical Infrastructure for Sensitive Documentation](#technical-infrastructure-for-sensitive-documentation)
- [Understanding Institutional Pressure Techniques](#understanding-institutional-pressure-techniques)
- [Incident Response Drills](#incident-response-drills)
- [Working with International Organizations](#working-with-international-organizations)
- [Ongoing Threat Assessment](#ongoing-threat-assessment)

## Understanding the Threat Environment

Conflict zone environments present threat models fundamentally different from corporate or personal security contexts. Your adversaries often include nation-state threat actors with significant resources, technical capability, and operational presence on the ground. The consequences of compromise extend beyond data theft to physical danger for you, your contacts, and local staff.

Before implementing any security tool or procedure, you must understand who is trying to harm you and what they want. A threat model is a document that answers three questions: What do I need to protect? Who wants it? What will they do to get it?

## Step 1: Identifying Critical Assets

Begin by listing everything that, if compromised, would cause harm. For human rights workers, assets typically include:

- **Source identities**: Names, locations, and any identifying information about people who provide information
- **Documentation**: Photos, videos, and written records of human rights violations
- **Communication metadata**: Who you contacted, when, and for how long
- **Location data**: Your movement patterns, safe house locations, and travel routes
- **Credentials**: Email passwords, VPN access, encryption keys, and device passcodes

Create a prioritized list ranked by the severity of harm if each asset is compromised. This ranking guides where to invest limited time and resources.

```python
# Example asset classification for threat modeling
assets = {
    "source_identities": {"severity": "critical", "mitigation_priority": 1},
    "documentation_raw": {"severity": "critical", "mitigation_priority": 1},
    "communication_content": {"severity": "high", "mitigation_priority": 2},
    "metadata": {"severity": "high", "mitigation_priority": 2},
    "device_credentials": {"severity": "medium", "mitigation_priority": 3}
}
```

## Step 2: Identifying and Profiling Adversaries

Your threat model must account for specific adversaries rather than generic "hackers." In conflict zones, relevant adversaries typically include:

**Nation-state actors** possess sophisticated capabilities including zero-day exploits, hardware implants, and the ability to compel cooperation from technology companies. They often operate with legal authority to conduct surveillance and can coordinate physical and digital operations.

**Local armed groups** may have different capabilities but often possess physical access advantages and may coerce local service providers, telecommunications companies, or internet service providers.

**Non-state actors** including criminal organizations and paramilitary groups may have technical capabilities acquired through commercial surveillance tools sold to governments worldwide.

For each adversary, assess their capability (technical resources), intent (motivation to target you), and opportunity (how they might reach you). Prioritize mitigations against adversaries with high capability, high intent, and high opportunity.

## Step 3: Analyzing Attack Vectors

With assets and adversaries defined, analyze how compromise could occur. Common attack vectors in conflict zones include:

### Network Interception

Telecommunications infrastructure is often controlled or compromised by hostile actors. Mobile networks, internet service providers, and Wi-Fi networks in conflict areas frequently perform deep packet inspection or full traffic capture.

**Mitigation**: Use end-to-end encrypted communication tools. Implement VPN tunnels that wrap all traffic in encryption, even when accessing seemingly secure services. Consider satellite communications as an alternative when terrestrial networks are compromised.

```bash
# Example: Creating an encrypted tunnel with WireGuard
# Install WireGuard: sudo apt install wireguard
# Generate keys: wg genkey | tee privatekey | wg pubkey > publickey

# Client configuration (wg0.conf)
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <vpn-server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Device Compromise

Physical device seizure or remote exploitation represents a primary threat vector. Devices may be seized during travel checkpoints, raids, or through social engineering.

**Mitigation**: Implement full disk encryption with keys derived from passphrases not stored on the device. Use ephemeral messaging systems that do not store message history locally. Prepare devices with plausible deniability using alternative operating systems.

```bash
# Example: Verifying full disk encryption status on Linux
sudo cryptsetup luksDump /dev/sda5 | grep -i "luks version"
# Check if LUKS container is unlocked
sudo cryptsetup luksUUID /dev/sda5
```

### Social Engineering

Phishing, pretexting, and impersonation attacks target human relationships rather than technical systems. Adversaries may pose as journalists, fellow activists, or international organization staff.

**Mitigation**: Establish out-of-band verification procedures for sensitive requests. Use dedicated communication channels for sensitive contacts. Verify identities through multiple independent channels before sharing sensitive information.

### Supply Chain Attacks

Adversaries may compromise software updates, hardware, or third-party services you rely on. This vector is particularly difficult to defend against and requires careful vendor selection.

**Mitigation**: Use reproducible builds to verify software integrity. Acquire hardware from trusted sources in friendly jurisdictions. Minimize dependence on third-party services for sensitive operations.

## Step 4: Implementing Practical Mitigations

Threat models are worthless without implementable mitigations. Focus on defenses that provide meaningful protection against your highest-priority adversaries while remaining usable in challenging field conditions.

### Communication Security

Signal remains the gold standard for encrypted messaging, but its security depends on proper configuration. Enable disappearing messages for all sensitive conversations. Verify safety numbers in person with trusted contacts.

```bash
# Signal CLI configuration example for headless operation
# Note: Signal is primarily mobile, but these principles apply
# - Enable disappearing messages (set to 24 hours or less)
# - Verify safety numbers through QR code exchange
# - Disable call answering on lock screen to prevent unauthorized access
```

For sensitive voice communication, consider dedicated encrypted voice solutions that support perfect forward secrecy and do not route through traditional telephone infrastructure.

### Data Handling

Implement a classification system for all collected information. Document evidence immediately upon collection using devices configured to minimize metadata retention.

```bash
# Example: Stripping EXIF metadata from images before transmission
# Using exiftool (install: apt install exiftool)
exiftool -all= -overwrite_original image.jpg
# Remove GPS, camera model, timestamps, and other identifying information
```

### Device Preparation

Prepare devices specifically for high-risk environments. This may include:

- Using devices with hardware kill switches for microphones and cameras
- Carrying decoy devices with innocuous content
- Using separate devices for different types of sensitive activity
- Implementing dead man's switches that automatically wipe devices after inactivity periods

## Step 5: Operational Security Procedures

Technical measures fail without operational procedures that account for human factors. Document security procedures and ensure all team members understand them:

Establish communication protocols that assume any channel may be compromised. Designate primary and backup communication methods with verification procedures for critical messages.

Create incident response plans for different compromise scenarios. Know in advance how to alert contacts, wipe devices, and safely evacuate if necessary.

Conduct regular security audits of your practices. Threat fields evolve rapidly in conflict zones, and what provided adequate protection last month may be insufficient today.

## Continuous Threat Model Maintenance

A threat model is not a one-time document but a living reference that requires regular updates. Review and update your threat model when:

- Operating in new geographic areas
- Taking on new types of documentation work
- Noticing new surveillance capabilities or tactics by adversaries
- After any security incident, near-miss or actual

Document your threat modeling process even if imperfect. The act of systematically thinking through assets, adversaries, and attack vectors significantly improves your security posture compared to ad-hoc security decisions.

## Building a Tiered Security Approach

Not all assets require the same level of protection. A tiered approach allocates resources efficiently:

**Tier 1 (Maximum Protection)**: Source identities and raw documentation
- Store on encrypted, air-gapped devices
- Never transmitted over public internet
- Backup only in secure locations controlled by trusted organizations
- Access limited to 1-2 people maximum

**Tier 2 (Strong Protection)**: Processed analysis and reports
- Encrypted at rest and in transit
- Can be shared with trusted international organizations
- No personally identifiable information about sources
- Transmission via Signal, not email

**Tier 3 (Standard Protection)**: Public-facing statements and published reports
- Standard website hosting and distribution
- No sensitive information
- Encrypted transport (HTTPS) only

Separating data by sensitivity tier prevents a single security breach from compromising everything. A device containing processed reports can be seized without exposing raw source identities if properly segregated.

## Technical Infrastructure for Sensitive Documentation

For organizations handling human rights documentation, infrastructure decisions have security implications:

**Cloud storage considerations**: Services like Google Drive and Dropbox are convenient but create centralized targets. Compromise of a single account exposes all stored materials. Consider instead:

```bash
# Self-hosted encrypted storage with Nextcloud
docker run -d --name nextcloud -v /data/nextcloud:/var/www/html -p 8080:80 nextcloud

# All data encrypted locally before uploading
gpg --symmetric --cipher-algo AES256 sensitive_report.pdf
# Verify encryption: file sensitive_report.pdf.gpg
```

Self-hosted storage using Nextcloud requires managing your own server infrastructure but eliminates a commercial vendor as a weakness. Encrypting files locally before uploading ensures even if the server is compromised, data remains protected.

**Off-the-grid backup strategy**: Critical evidence should exist in locations completely independent of internet-connected systems. Consider:

1. **Encrypted hard drives in geographically distributed locations**: Send encrypted drives to trusted partner organizations in different countries. Geographically dispersed backups mean seizure of one location doesn't eliminate all evidence.

2. **Dead drops for critical documentation**: Arrange secure locations (unattended) where encrypted drives are deposited and picked up by trusted contacts. Eliminates surveillance of handoff meetings.

3. **Paper backup of critical evidence**: Print high-resolution photos of key evidence on archival-quality paper stored in secure locations. Digital systems fail, but properly stored paper lasts decades.

## Understanding Institutional Pressure Techniques

Sophisticated state actors don't always attack directly. They apply pressure through institutional channels:

**Visa denial and travel restriction**: Journalists and human rights workers find themselves unable to travel, attend conferences, or meet with donors. This isn't a security vulnerability you can patch—it's institutional pressure requiring political response.

**Banking discrimination**: Organizations get flagged in banking systems, making wire transfers impossible or extremely slow. International funding becomes inaccessible despite no formal sanctions.

**Regulatory harassment**: Governments demand documentation, threaten closure, increase compliance overhead. Legitimate but designed to exhaust resources.

**Device interdiction**: Laptops are "lost" in transit, phones go missing, equipment gets confiscated. Mitigate by:
- Never travel with equipment containing sensitive data
- Use temporary devices that can be abandoned if seized
- Carry decoys that can be surrendered to authorities
- Ship sensitive equipment separately rather than carrying

## Incident Response Drills

A security plan fails if no one knows how to execute it during actual crisis. Run drills quarterly:

**Drill 1: Device compromise scenario**: Someone reports suspicious activity. Assume the device is compromised. Can you wipe it cleanly? Can you restore your communications within 2 hours? Do team members know the emergency contact procedure?

**Drill 2: Communication interception**: Assume your primary communication channel is compromised. Activate backup channel. Can all team members reach you through the secondary method? Does the secondary channel actually work or was it never tested?

**Drill 3: Arrest scenario**: A team member is detained. Have you documented their recovery phrases? Can someone else access their accounts to disable them if needed? Have you prepared a legal response? (Many organizations have retainer agreements with lawyers who specialize in activist defense.)

Document drill results. Each drill reveals gaps in your procedures. Fix gaps before actual emergencies.

## Working with International Organizations

Many human rights workers coordinate with UN agencies, international NGOs, and diplomatic missions. These relationships create both security and vulnerability:

**Secure communication with international bodies**: UN agencies and embassy staff use varied security standards. Never assume their systems meet your security requirements:

```bash
# Before sharing sensitive information:
# 1. Ask about their encryption and retention policies
# 2. Request written confirmation of their security procedures
# 3. Consider all communications with them may be accessible to their government
# 4. Use intermediary organizations with documented security if direct contact is unsafe
```

UN offices and embassies in conflict zones may be targets themselves or may have been compromised by hostile actors. Verify independently that the person you're communicating with is actually who they claim to be. Use out-of-band verification (phone call to known number, in-person meeting with verified identity).

## Ongoing Threat Assessment

Conflict zones evolve. Threats that existed last month may be replaced by new threats this month:

**Monitor changing surveillance tactics**: If adversaries shift from physical surveillance to digital interception, your mitigations need to shift. An organization focused on preventing physical device seizure while ignoring network eavesdropping is unprepared.

**Track adversary capability changes**: When new surveillance tools become available, update your threat model. Commercial spyware platforms like NSO's Pegasus or Candiru represent significant capability jumps.

**Adjust for organizational evolution**: As your organization grows, your threat model changes. A solo researcher has different threats than a coordinated network of 50 people. Revisit your model as team size changes.

**Update after near-misses**: If someone almost gets caught, investigate what went wrong. Did your procedures fail, or did they reveal new threat vectors? Update accordingly.

## Frequently Asked Questions

**How long does it take to complete this setup?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Threat Model Assessment For High Risk Journalist In Hostile](/privacy-tools-guide/threat-model-assessment-for-high-risk-journalist-in-hostile-/)
- [Threat Model For Corporate Whistleblower Protecting Evidence](/privacy-tools-guide/threat-model-for-corporate-whistleblower-protecting-evidence/)
- [Threat Model For Source Communicating With Journalist](/privacy-tools-guide/threat-model-for-source-communicating-with-journalist-anonym/)
- [Threat Model For Political Dissident In Surveillance State](/privacy-tools-guide/threat-model-for-political-dissident-in-surveillance-state-2/)
- [Threat Model For Activist In Authoritarian Country Digital](/privacy-tools-guide/threat-model-for-activist-in-authoritarian-country-digital-s/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
