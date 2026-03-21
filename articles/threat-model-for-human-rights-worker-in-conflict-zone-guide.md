---
layout: default
title: "Threat Model For Human Rights Worker In Conflict Zone Guide"
description: "A threat modeling guide for human rights workers in conflict zones. Learn to identify assets, analyze adversaries, and implement"
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /threat-model-for-human-rights-worker-in-conflict-zone-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Human rights workers in conflict zones face nation-state and paramilitary threats with capabilities including physical surveillance, device seizure with forensic extraction, social network analysis, and sophisticated network eavesdropping. Protect evidence of human rights abuses using Tails Linux for documentation work, Signal encrypted communications, encrypted storage vaults for sensitive documents, and regular device wipes. Establish secure communication with international organizations, separate your work device from personal phone, use dead drops or air-gapped document transfer, and implement strict access controls on victim testimony. This guide provides a practical threat modeling framework covering asset identification, adversary capability analysis, and actionable mitigations tailored to conflict zone environments.

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

Conduct regular security audits of your practices. Threat landscapes evolve rapidly in conflict zones, and what provided adequate protection last month may be insufficient today.

## Continuous Threat Model Maintenance

A threat model is not an one-time document but a living reference that requires regular updates. Review and update your threat model when:

- Operating in new geographic areas
- Taking on new types of documentation work
- Noticing new surveillance capabilities or tactics by adversaries
- After any security incident, near-miss or actual

Document your threat modeling process even if imperfect. The act of systematically thinking through assets, adversaries, and attack vectors significantly improves your security posture compared to ad-hoc security decisions.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
