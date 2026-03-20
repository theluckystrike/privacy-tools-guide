---

layout: default
title: "Turkey Journalist Digital Safety Guide: Protecting Sources and Communications from Government Surveillance"
description: "A technical guide for Turkish journalists on securing communications, protecting sources, and mitigating government surveillance threats using open-source tools and advanced security practices."
date: 2026-03-16
author: theluckystrike
permalink: /turkey-journalist-digital-safety-guide-protecting-sources-an/
categories: [security, guides]
---

{% raw %}

Journalists operating in Turkey face sophisticated surveillance threats from state-level actors. The country's history of internet censorship, social media blocking, and targeted arrests of reporters demands a proactive approach to digital security. This guide provides practical, developer-focused strategies for protecting sources and communications against government surveillance.

## Understanding the Threat Model

Turkish authorities employ deep packet inspection (DPI), mandatory data retention laws, and periodic social media restrictions. Internet service providers (ISPs) collaborate with government requests, and journalists have been prosecuted based on communication metadata. Your threat model must account for:

- Traffic analysis identifying who communicates with whom
- ISP-level monitoring of unencrypted traffic
- Device seizure and forensic analysis
- Social media platform subpoenas and account takeovers
- Phishing campaigns targeting journalists

## Secure Communications Architecture

### End-to-End Encrypted Messaging

Avoid standard SMS and unencrypted messaging apps. Signal remains the baseline recommendation, but power users should configure additional layers:

```bash
# Verify Signal installation integrity on Linux
# Download the Signal signing key
wget -O- https://updates.signal.org/desktop/apt/keys.asc | gpg --dearmor > signal-desktop.gpg
sudo install -o root -g root -m 644 signal-desktop.gpg /usr/share/keyrings/
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/signal-desktop.gpg] https://updates.signal.org/desktop/apt xenial main" | \
  sudo tee /etc/apt/sources.list.d/signal-xenial.list
sudo apt update && sudo apt install signal-desktop
```

Configure Signal to:
- Enable disappearing messages with 5-minute expiration
- Disable link previews to prevent metadata leakage
- Register a separate number used only for sensitive communications

### Encrypted Email with ProtonMail and GPG

For source communications requiring email, combine ProtonMail's zero-access encryption with GPG for additional protection:

```bash
# Generate a dedicated journalism GPG key
gpg --full-generate-key
# Select RSA 4096-bit key
# Use a dedicated email like journalist@protonmail.com
# Set expiration to 1 year

# Export your public key for source communication
gpg --armor --export your-journalism-email@example.com > journalist-public.asc

# Encrypt sensitive documents before attachment
gpg --encrypt --recipient source@secure-email.com --armor sensitive-document.asc
```

Sources should generate their own GPG keys. Never store private keys on devices that could be seized—use YubiKey or similar hardware security modules.

## Network-Level Protection

### Tor and Obfs4 Bridges

Turkey periodically blocks Tor bridges. Use obfs4 bridges to circumvent censorship:

```bash
# Install Tor Browser
# Configure bridges in Tor Browser settings
# Use obfs4 bridges from https://bridges.torproject.org/

# For command-line Tor usage (advanced)
sudo apt install tor
sudo nano /etc/tor/torrc

# Add these lines for obfs4 bridges:
# UseBridges 1
# Bridge obfs4 192.0.2.1:443 certificate=... iat-mode=2
# Bridge obfs4 192.0.2.2:443 certificate=... iat-mode=2
```

Rotate bridges regularly and consider using meek tactics for additional obfuscation.

### DNS Configuration

Avoid DNS leaks that can reveal browsing activity:

```bash
# Configure systemd-resolved for encrypted DNS
sudo nano /etc/systemd/resolved.conf

# Add:
[Resolve]
DNS=9.9.9.9#dns.quad9.net 2620:fe::fe#dns.quad9.net
DNSSEC=yes
DNSOverTLS=yes

sudo systemctl restart systemd-resolved
```

For mobile devices, use private DNS (DoH) with a provider like Quad9 or Cloudflare's 1.1.1.1.

## Device Security and Seizure Protection

### Full Disk Encryption

Enable LUKS encryption on Linux or FileVault on macOS. For Turkish journalists, consider plausible deniability tools like VeraCrypt hidden volumes:

```bash
# Create VeraCrypt hidden volume (command-line)
veracrypt -c --size=500M --password=outerpass --hash=SHA-512 --encryption=AES --filesystem=FAT -p /dev/sdX

# Create hidden volume within
veracrypt -c --size=200M --password=hiddenpass --hash=SHA-512 --encryption=AES --filesystem=FAT -p /dev/sdX --hidden
```

Never reveal the outer volume password during device seizure.

### Air-Gapped Source Document Storage

Store highly sensitive documents on air-gapped machines:

1. Use an old laptop with no network interface
2. Install Tails or Qubes OS
3. Transfer documents via encrypted USB using LUKS
4. Wipe the USB after transfer using `shred -v /dev/sdX`

## Metadata Stripping and Verification

### Document Sanitization

Before publishing, strip metadata from documents:

```bash
# Install mat2 (metadata anonymisation tool)
sudo apt install mat2

# Clean individual files
mat2 sensitive-document.pdf

# Batch clean directory
for file in *.pdf *.docx *.jpg; do mat2 "$file"; done

# Use exiftool for advanced metadata removal
sudo apt install libimage-exiftool-perl
exiftool -all= image.jpg
```

### Screenshot Verification

When receiving sensitive screenshots, verify they haven't been tampered with:

```bash
# Generate hash for source verification
sha256sum screenshot.png > screenshot.sha256

# Source sends you hash via separate channel
# Verify integrity
sha256sum -c screenshot.sha256
```

## Operational Security Habits

### Separation of Identities

Maintain strict separation between:

- Personal accounts (WhatsApp, personal email)
- Professional accounts (Signal, work email)
- Sensitive source communication (dedicated devices, separate numbers)

Never log into sensitive accounts from personal devices or public networks.

### Secure Deletion

Standard file deletion does not remove data. Use secure deletion tools:

```bash
# Shred files with 35-pass overwrite (military-grade)
shred -v -n 35 sensitive-file.pdf

# Wipe free space
dd if=/dev/zero of=/tmp/wipefile bs=1M
rm /tmp/wipefile
# Or use: cat /dev/zero > /tmp/wipe; rm /tmp/wipe

# For SSD/flash storage, use hdparm or manufacturer tools
# Note: SSDs with TRIM may not allow secure deletion
```

### Regular Security Audits

Implement monthly security reviews:

1. Check login locations for all accounts
2. Rotate GPG keys and passwords
3. Verify Signal safety numbers with sources
4. Review and revoke unnecessary OAuth permissions
5. Update all software to latest versions

## Emergency Protocols

Prepare for potential device seizure:

1. **Remote wipe capability**: Configure Find My Device (iOS) or Find My Device (Android) with remote wipe
2. **Dead man's switch**: Use a timed encrypted message service that releases information if you don't check in
3. **Source contact schedule**: Establish regular check-in times with sources
4. **Legal contacts**: Have digital rights lawyer contact information readily available

## Conclusion

Digital security for Turkish journalists requires layered defenses spanning communications, network access, device protection, and operational habits. No single tool provides complete protection—adopt a defense-in-depth strategy and regularly audit your security posture. Stay informed about current censorship methods and maintain backup communication channels with sources.

Remember: the goal is not perfect security but making surveillance sufficiently expensive and time-consuming to protect your sources and your work.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
