---
---
layout: default
title: "Privacy Setup for Confidential Informant"
description: "A practical guide to privacy setup for confidential informant protection. Learn technical strategies, tools, and code examples to protect your identity"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /privacy-setup-for-confidential-informant-protecting-identity/
categories: [guides, security]
reviewed: true
intent-checked: true
voice-checked: true
score: 9
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Protecting your identity as a confidential informant requires a multi-layered approach combining operational security, secure communication channels, and careful digital hygiene. This guide provides practical technical strategies for developers and power users who need privacy setup for confidential informant scenarios.

# 3.
- **Communication verification - Out-of-band**: key verification performed - Safety numbers verified in person - Challenge-response authentication used 5.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced Threat Modeling for CI Protection](#advanced-threat-modeling-for-ci-protection)
- [Tails OS Configuration for Maximum Anonymity](#tails-os-configuration-for-maximum-anonymity)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand the Threat Model

Before implementing any privacy setup, you must understand what you're protecting against. Adversaries range from local surveillance to sophisticated state-level actors with access to ISP records, device exploits, and metadata analysis capabilities. Your threat model determines which tools and techniques are necessary.

Key threats include:
- **Metadata collection**: Even encrypted communications reveal who contacted whom and when
- **Device compromise**: Malware can exfiltrate data before encryption
- **Social engineering**: Reconnaissance through your digital footprint
- **Traffic analysis**: Pattern recognition on network traffic

### Step 2: Device Isolation and Air-Gapping

The foundation of any privacy setup for confidential informant protection is separating your sensitive activities from your daily driver devices. Air-gapping means keeping a dedicated device offline for any work involving identifying information.

Consider a Raspberry Pi configured as a dedicated air-gapped machine:

```bash
# Install a minimal Debian-based system
curl -fsSL https://raspi.debian.net/tested/raspi_4_bookworm.img.xz | xz -d | sudo dd of=/dev/sdX bs=4M status=progress

# Disable all network interfaces post-installation
sudo systemctl mask NetworkManager.service
sudo systemctl mask wpa_supplicant.service
```

This device never connects to any network. Transfer files using encrypted USB drives with fresh formatting between uses. Label these devices ambiguously to avoid attracting attention during travel.

### Step 3: Secure Operating System Configuration

For devices that must remain online, use a privacy-focused distribution with hardened defaults. Qubes OS provides strong isolation through its Xen-based virtualization, allowing you to run separate domains for different activities.

Essential hardening steps for any Linux distribution:

```bash
# Disable CPU frequency scaling (prevents timing attacks)
sudo echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Disable swap to prevent data written to disk
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Install and configure firejail for application sandboxing
sudo apt install firejail
firejail --private --net=none firefox

# Configure system DNS over TLS
echo -e "[Resolve]\nDNSOverTLS=opportunistic" | sudo tee /etc/systemd/resolved.conf.d/dns.conf
sudo systemctl restart systemd-resolved
```

### Step 4: Encrypted Communications

When communicating with handlers or contacts, use end-to-end encrypted platforms with forward secrecy. Signal remains the gold standard for secure messaging, but verify registration keys through a separate channel.

For developers building secure communication tools, consider these libraries:

```python
# Example using Signal Protocol library
from signal_protocol import curve, ratchet, protocol
import libsodium

# Generate identity key pair
identity_key_pair = curve.generate_key_pair()
registration_id = libsodium.randombytes_uniform(65535)

# Pre-key bundle for asynchronous key exchange
pre_key_bundle = {
    'identity_key': identity_key_pair.public,
    'registration_id': registration_id,
    'pre_key_id': libsodium.randombytes_uniform(65535),
    'pre_key': curve.generate_key_pair(),
    'signed_pre_key': curve.generate_key_pair(),
    'signed_pre_key_signature': None  # Sign with identity key
}
```

Implement disappearing messages with short timers, and verify session fingerprints out-of-band.

### Step 5: Metadata Protection

Metadata often reveals more than content. Your phone routinely logs cell tower connections, WiFi networks, and GPS coordinates. Disable these features:

```bash
# Android: Disable location services via ADB
adb shell settings put secure location_mode 0

# iOS: Disable significant Locations
# Settings > Privacy > Location Services > System Services > Significant Locations = Off

# Linux: Prevent WiFi geolocation
sudo systemctl mask geoclue.service
sudo chmod -x /usr/libexec/geoclue-2.0/demo/agent
```

Use Tor for all network traffic when online. Configure your applications to use Tor socks proxy:

```bash
# Configure curl to use Tor
curl --socks5 localhost:9050 https://check.torproject.org/api/ip

# Configure git to use Tor
git config --global http.proxy socks5h://localhost:9050
git config --global https.proxy socks5h://localhost:9050
```

### Step 6: Digital Footprint Management

Your existing digital presence can compromise your privacy setup. Conduct an audit:

1. **Search yourself** across search engines for old accounts
2. **Request data deletion** under GDPR Article 17 (right to erasure)
3. **Remove EXIF data** from photos before sharing

```bash
# Strip EXIF data using exiftool
exiftool -all= -overwrite_original sensitive_photo.jpg

# Batch process directory
find ./photos -type f -exec exiftool -all= {} \;
```

Create completely separate identities with no linked information. Use distinct email addresses, phone numbers (burner SIMs), and payment methods. Never cross-contaminate these identities.

### Step 7: Operational Security Habits

Technical tools fail without consistent operational practices:

- **Device locking**: Enable full disk encryption with keys stored on separate hardware tokens
- **Screen discipline**: Use privacy screens in public, and never work on sensitive documents where cameras might capture them
- **Memory protection**: Reboot regularly to clear sensitive data from RAM
- **Paper notes**: Keep minimal written records; use destructible notebooks for any temporary notes

```bash
# Secure memory wipe demonstration (Linux)
sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
sudo cryptsetup luksClose encrypted_volume
```

### Step 8: Plan Incident Response Plan

Despite precautions, compromise may occur. Prepare:

1. **Evidence destruction**: Have protocols for immediate data wiping
2. **Exfiltration paths**: Know safe houses and communication backups
3. **Legal preparation**: Understand your rights regarding compelled disclosure
4. **Contact protocols**: Establish dead drops or scheduled check-ins with handlers

Document your security setup but store that documentation separately from the devices themselves.

## Advanced Threat Modeling for CI Protection

Develop precise threat models based on adversary capabilities:

```python
# Threat model assessment framework
class ConfidentialInformantThreatModel:
    def __init__(self):
        self.threats = {
            "local_law_enforcement": {
                "capabilities": [
                    "Physical device access",
                    "Local ISP traffic monitoring",
                    "Vehicle surveillance",
                    "Financial transaction tracking"
                ],
                "mitigation": [
                    "Air-gapping critical device",
                    "Regular device rotation",
                    "Physical counter-surveillance awareness"
                ]
            },
            "federal_agency": {
                "capabilities": [
                    "National backbone monitoring",
                    "0-day exploits",
                    "Device wiretapping",
                    "Financial account access"
                ],
                "mitigation": [
                    "Tor for all network activity",
                    "Compartmentalized identities",
                    "Paper-based dead drops for critical info"
                ]
            },
            "sophisticated_criminal_org": {
                "capabilities": [
                    "Technical exploitation",
                    "Social engineering",
                    "Surveillance countermeasures",
                    "Corruption of gatekeepers"
                ],
                "mitigation": [
                    "Operational security discipline",
                    "Multiple independent channels",
                    "Regular threat re-assessment"
                ]
            }
        }

    def assess_threat_level(self, adversary):
        if adversary not in self.threats:
            return None
        return self.threats[adversary]

# Usage
model = ConfidentialInformantThreatModel()
threat = model.assess_threat_level("federal_agency")
print(f"Mitigations: {threat['mitigation']}")
```

### Step 9: Multi-Device Isolation Strategy

Implement strict compartmentalization:

```bash
# Device 1: Air-gapped writing device
# - Debian on offline laptop
# - No WiFi/Bluetooth hardware
# - External USB keyboard (no internal input devices)
# - Used only for drafting sensitive documents

# Device 2: Tor communication device
# - Live USB running Tails OS
# - Reboots fresh on each session
# - Pre-loaded with Signal configuration
# - Only connects through Tor + VPN chain

# Device 3: Clean device for public activities
# - Android phone with minimal apps
# - Regular web browsing, email
# - Completely separate identity
# - Never used for sensitive communications

# Device 4: Backup recovery device
# - Encrypted recovery keys for all other devices
# - Stored in physically secure location
# - Accessed only in emergency

# Test device compartmentalization
#!/bin/bash
# Verify devices don't communicate
for device in air-gapped-laptop tor-device android-phone; do
    echo "Testing $device isolation..."
    arp-scan -l | grep -v $device
    # Should find no common networks
done
```

## Tails OS Configuration for Maximum Anonymity

For high-risk CI operations:

```bash
#!/bin/bash
# Tails OS hardening for CI protection

# 1. Create persistent volume on USB
# This survives reboots and maintains configuration

# 2. Configure automatic bridges for Tor
# Prevents ISP from detecting Tor usage
# Settings > Tor Connection > Automatic Bridges

# 3. Set hostname to maximum entropy
# Instead of "tails-user"
sudo hostnamectl set-hostname $(openssl rand -hex 4)

# 4. Enable firewall with minimal exceptions
sudo systemctl enable ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 5. Disable ICMP to prevent ping detection
sudo iptables -A INPUT -p icmp -j DROP

# 6. Configure AppArmor for application confinement
sudo systemctl enable apparmor
sudo aa-enforce /etc/apparmor.d/*

# 7. Set aggressive swap configuration
echo "vm.swappiness=0" | sudo tee -a /etc/sysctl.conf

# 8. Disable all unnecessary services
sudo systemctl disable bluetooth
sudo systemctl disable cups
sudo systemctl disable avahi-daemon
```

### Step 10: Dead Drop Communication Protocols

Implement truly offline communication channels:

```bash
#!/bin/bash
# Dead drop setup for critical CI communications

# Physical dead drop: Pre-arranged location for message exchange
# 1. Pick obscure but accessible location
# 2. Use signal items (chalk mark, stone placement) to indicate new message
# 3. Messages in sealed envelope, coded if necessary
# 4. Check frequency: Every Sunday 3 PM, or on agreed signal

# Example dead drop signal system
# - White stone on fence: Check location for message
# - Two white stones: Do not approach, area under surveillance
# - Dead drop location: Tree root marker, 10 paces north of telephone pole

# Digital dead drop using compromised accounts
# 1. Create fake email account (no real personal info)
# 2. Exchange drafts without sending
# 3. Both parties access same account, read drafts, delete after reading
# 4. Leaves no transmission records
# 5. Credentials provided out-of-band (spoken, written, memorized)

# Example Gmail dead drop
# Username: observer.field.report@gmail.com
# Use: Settings > Forwarding and POP/IMAP > Disable forwarding
# Creates draft messages
# CI handler and CI both access same account from secure networks
# Review drafts, delete after reading
```

### Step 11: Counter-Surveillance Awareness

Operational security extends beyond technology:

```bash
#!/bin/bash
# Counter-surveillance checklist

# Physical security
- [ ] Vary routes to communication locations
- [ ] Check for surveillance vehicles (repeated license plates)
- [ ] Avoid fixed schedule patterns
- [ ] Conduct counter-surveillance of your own patterns
- [ ] Use public transportation (harder to follow than personal vehicle)
- [ ] Identify escape routes from meeting locations

# Digital security
- [ ] Assume all wireless networks are monitored
- [ ] Never use personal identifying information in emergency accounts
- [ ] Rotate devices regularly (if possible)
- [ ] Check device for physical tampering (micro-cameras, etc.)
- [ ] Monitor device battery drain (indicator of malware)

# Communication security
- [ ] Verify handler identity before sharing sensitive info
- [ ] Use predetermined challenge-response phrases
- [ ] Abort communication if anything feels wrong
- [ ] Never discuss methods in recorded communications
- [ ] Assume voice communications are recorded

# Financial security
- [ ] Pay cash for SIM cards, not credit card
- [ ] Use separate payment method for each distinct identity
- [ ] Be alert to unusual financial scrutiny
- [ ] Maintain separate bank account for handler payments (if applicable)
```

### Step 12: Evidence Preservation and Legal Protection

Document your security practices for legal proceedings:

```bash
#!/bin/bash
# Security practice documentation

cat > ~/ci-security-practices.txt <<'EOF'
Date Documented: $(date)

SECURITY PRACTICES IMPLEMENTED:
1. Device isolation
   - Air-gapped writing device for sensitive materials
   - Tor-only communication device
   - Public identity device completely separate

2. Encryption standards
   - All data encrypted with AES-256
   - All communications end-to-end encrypted
   - Disk full-disk encrypted

3. Key management
   - Keys not stored on networked devices
   - Key escrow with trusted party for emergency access
   - Regular key rotation

4. Communication verification
   - Out-of-band key verification performed
   - Safety numbers verified in person
   - Challenge-response authentication used

5. Operational discipline
   - No metadata cross-contamination
   - Regular device replacement
   - Physical counter-surveillance awareness

This document establishes that security practices met
industry standards for information protection.
EOF

# Encrypt and secure this document
gpg --symmetric --cipher-algo AES256 ~/ci-security-practices.txt
shred -vfz -n 10 ~/ci-security-practices.txt
chmod 600 ~/ci-security-practices.txt.gpg
```

### Step 13: Malware Detection and Response

Identify if your systems have been compromised:

```bash
# Monitor for common CI compromise indicators

# 1. File system changes
cd ~/Documents
find . -type f -mtime -1  # Modified in last 24 hours
# Unexpected changes suggest malware

# 2. Process monitoring
ps auxww | grep -E "ssh|nc|curl|wget|telnet"
# Verify all processes are user-initiated

# 3. Network connections
netstat -tupln | grep -E ":[0-9]+.*ESTABLISHED"
# Check for unexpected outbound connections

# 4. Port monitoring
netstat -tulpn | grep LISTEN
# Look for unexpected listening ports

# 5. Cron job verification
crontab -l
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u $user -l 2>/dev/null
done
# Malware often installs cron persistence

# 6. Sudoers verification
sudo cat /etc/sudoers
sudo cat /etc/sudoers.d/*
# Check for unauthorized sudo entries

# 7. If compromise suspected:
# - Immediately shutdown (hard shutdown, don't sync)
# - Do not use system for further communications
# - Recover with fresh operating system
# - Notify handler through alternative channel
```

### Step 14: Recovery and Continuity Planning

Prepare for worst-case scenarios:

```bash
# Emergency continuity procedures

# 1. Device compromise recovery
# - Have pre-positioned backup CI device ready to activate
# - Encrypted credentials stored separately
# - Activation procedure: Decrypt credentials, establish Tor connection

# 2. Communication re-establishment
# - Secondary handler contact method pre-arranged
# - Agreed signal to indicate compromise
# - Alternative meeting locations pre-scouted

# 3. Legal support activation
# - Attorney contact information memorized (not written)
# - Legal funds in protected account
# - Instructions for attorney to follow if you're detained

# 4. Physical safety
# - Safe house location known to trusted contact
# - Bug-out bag maintained at secure location
# - Travel documents (if applicable) stored separately

# 5. Post-compromise review
# - All devices assumed compromised
# - All previous contacts potentially exposed
# - Full threat re-assessment necessary
# - New operational security posture required
```

The CI protection setup must be continuously updated as threats evolve and new vulnerabilities are discovered. Regular security audits and threat reassessments are non-negotiable for maintaining protection.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to confidential informant?**

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

- [Privacy Setup For Immigration Activist Protecting Undocument](/privacy-tools-guide/privacy-setup-for-immigration-activist-protecting-undocument/)
- [How to Use Tails OS for Maximum Privacy Complete Setup Guide](/privacy-tools-guide/how-to-use-tails-os-for-maximum-privacy-complete-setup-guide/)
- [Privacy by Design Principles: A Practical Guide](/privacy-tools-guide/privacy-by-design-principles-practical-guide/)
- [Privacy Fatigue Solutions: How to Make Privacy Easier Guide](/privacy-tools-guide/privacy-fatigue-solutions-how-to-make-privacy-easier-guide/)
- [Privacy Setup For Abuse Hotline Worker Protecting Caller](/privacy-tools-guide/privacy-setup-for-abuse-hotline-worker-protecting-caller-inf/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
