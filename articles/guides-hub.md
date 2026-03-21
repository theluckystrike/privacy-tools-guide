---
layout: default
title: "Guides — Complete Guide"
description: "Browse every privacy and security guide in one place — from VPN setup and password manager reviews to encrypted messaging comparisons and browser"
date: 2026-03-15
last_modified_at: 2026-03-15
permalink: /guides-hub/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

# Guides — Complete Guide Hub

Browse every privacy and security guide in one place — from VPN setup and password manager reviews to encrypted messaging comparisons and browser fingerprinting defense. Each article below is technically reviewed and scored for depth.

- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [VPN Warrant Canary: What It Means and Why It Matters](/privacy-tools-guide/vpn-warrant-canary-what-it-means/)
- [Best VPN for Linux Desktop: A Developer Guide](/privacy-tools-guide/best-vpn-for-linux-desktop/)
- [Firefox Strict Tracking Protection vs Custom: A.](/privacy-tools-guide/firefox-strict-tracking-protection-vs-custom/)
- [Best Password Manager for Small Business: A Developer Guide](/privacy-tools-guide/best-password-manager-for-small-teams-2026/)
- [ProtonMail Security Model Explained: A Technical Deep-Dive](/privacy-tools-guide/protonmail-security-model-explained/)
- [Best Secure Video Calling App 2026: A Technical Guide](/privacy-tools-guide/best-secure-video-calling-app-2026/)
- [Bitwarden Premium Worth the Cost 2026: A Developer's.](/privacy-tools-guide/bitwarden-premium-worth-the-cost-2026/)
- [Best VoIP App with Encryption 2026: A Developer and Power User Guide](/privacy-tools-guide/best-voip-app-with-encryption-2026/)
- [How Browser Fingerprinting Works Explained: A Technical.](/privacy-tools-guide/how-browser-fingerprinting-works-explained/)
- [Best Browser for Developers Privacy 2026: A Technical Guide](/privacy-tools-guide/best-browser-for-developers-privacy-2026/)
- [How Secure is Telegram Secret Chat Mode: A Technical Deep Dive](/privacy-tools-guide/how-secure-is-telegram-secret-chat-mode/)
- [Best VPN for Streaming Hulu Abroad](/privacy-tools-guide/best-vpn-for-streaming-hulu-abroad/)
- [Best Encrypted Notes App 2026: A Developer's Guide](/privacy-tools-guide/best-encrypted-notes-app-2026/)
- [Best Browser for Avoiding Google Tracking: A Developer Guide](/privacy-tools-guide/best-browser-for-avoiding-google-tracking/)
- [Best Private Email with Zero Knowledge: A Developer Guide](/privacy-tools-guide/guides-hub/)
- [Threema vs Signal Privacy Comparison 2026: A Technical Analysis](/privacy-tools-guide/threema-vs-signal-privacy-comparison-2026/)
- [Browser Password Manager vs Dedicated App: A Developer Comparison](/privacy-tools-guide/browser-password-manager-vs-dedicated-app/)
- [Best Encrypted File Sharing Service 2026: A Developer's.](/privacy-tools-guide/best-encrypted-file-sharing-service-2026/)
- [Brave vs Safari Privacy Comparison 2026: A Developer Guide](/privacy-tools-guide/brave-vs-safari-privacy-comparison-2026/)
- [VPN over Tor vs Tor over VPN: A Technical Comparison](/privacy-tools-guide/vpn-over-tor-vs-tor-over-vpn/)
- [XMPP OMEMO Encryption Setup Guide: Secure Your Instant.](/privacy-tools-guide/xmpp-omemo-encryption-setup-guide/)
- [Best Secure Group Chat App 2026: A Technical Guide](/privacy-tools-guide/best-secure-group-chat-app-2026/)
- [1Password Watchtower Feature Review: A Developer's Guide](/privacy-tools-guide/1password-watchtower-feature-review/)
- [Best Password Manager for Developers: A Practical Guide](/privacy-tools-guide/best-password-manager-for-developers/)
- [Signal App Disappearing Messages Guide: Technical Configuration for Power Users](/privacy-tools-guide/signal-app-disappearing-messages-guide/)
- [Best Password Manager for Small Teams in 2026: A Developer Guide](/privacy-tools-guide/best-password-manager-for-small-teams-2026/)
- [ProtonMail Two-Factor Authentication Guide: Secure Your Email](/privacy-tools-guide/protonmail-two-factor-authentication-guide/)
- [VPN Connection Drops Troubleshooting Guide](/privacy-tools-guide/vpn-connection-drops-troubleshooting-guide/)


## Verifying VPN Leak Protection

Before trusting any VPN for sensitive browsing, verify that DNS and WebRTC leaks are absent.

```bash
# Check for DNS leaks via CLI
curl -s https://am.i.mullvad.net/json | python3 -m json.tool

# Test WebRTC leak (requires browser extension or:)
# Open https://browserleaks.com/webrtc with VPN active
# Your VPN IP should appear, not your real IP

# Confirm kill-switch is active on Linux
iptables -L OUTPUT -n | grep -E "DROP|REJECT"
```

Run these tests immediately after connecting and again after a brief network disruption. A good kill-switch blocks all traffic when the tunnel drops, not just new connections.

## Split Tunneling Configuration

Split tunneling lets you route only specific apps through the VPN while leaving other traffic on your regular connection.

```bash
# Mullvad CLI split tunneling example
mullvad split-tunnel add /usr/bin/curl
mullvad split-tunnel list

# On Linux with NetworkManager, exclude a subnet:
nmcli connection modify "VPN-Name"   ipv4.routes "10.0.0.0/8"   ipv4.never-default yes
```

Use split tunneling for high-bandwidth streaming while keeping your browser and messaging apps tunneled. Never split-tunnel password managers or banking apps.

## Browser Privacy Hardening Essentials

Browser configuration is one of the highest-impact privacy improvements you can make. Most tracking happens through your browser, making it the first line of defense.

### Firefox Privacy Configuration via about:config

Firefox offers granular privacy controls through its `about:config` interface:

```
privacy.resistFingerprinting = true
privacy.trackingprotection.enabled = true
network.cookie.cookieBehavior = 5
dom.security.https_only_mode = true
media.peerconnection.enabled = false
geo.enabled = false
```

Set `privacy.resistFingerprinting` to true first -- it changes dozens of settings simultaneously, making your browser harder to uniquely identify. Disabling WebRTC via `media.peerconnection.enabled` prevents IP leaks even when using a VPN.

### Recommended Browser Extensions

| Extension | Purpose | Privacy Impact |
|-----------|---------|---------------|
| uBlock Origin | Ad and tracker blocking | High -- blocks most third-party trackers |
| Privacy Badger | Learning-based tracker blocking | Medium -- catches trackers uBlock misses |
| HTTPS Everywhere | Force HTTPS connections | Medium -- prevents downgrade attacks |
| Cookie AutoDelete | Remove cookies after tab closes | High -- limits persistent tracking |
| Decentraleyes | Local CDN emulation | Low-Medium -- reduces CDN tracking |

Install uBlock Origin first -- it provides the largest single privacy improvement.

## Password Manager Selection Criteria

A password manager is non-negotiable for security. When evaluating options, consider these technical factors:

- **Encryption algorithm**: AES-256 with PBKDF2 or Argon2 key derivation
- **Zero-knowledge architecture**: The provider should never have access to your vault contents
- **Audit history**: Look for recent third-party security audits published publicly
- **Export capability**: You should always be able to export your data in a standard format
- **2FA support**: TOTP at minimum, FIDO2/WebAuthn preferred

### Quick Comparison

| Manager | Encryption | Zero-Knowledge | Open Source | Audited |
|---------|-----------|----------------|-------------|---------|
| Bitwarden | AES-256/PBKDF2 | Yes | Yes | Yes (2023) |
| 1Password | AES-256/SRP | Yes | Partial | Yes (2022) |
| KeePassXC | AES-256/Argon2 | Yes (local) | Yes | Community |
| Proton Pass | AES-256/Argon2 | Yes | Yes | Yes (2024) |

For most users, Bitwarden offers the best combination of security, usability, and cost. Self-hosting Bitwarden via Vaultwarden gives you full control over your credential storage.

## Email Privacy Fundamentals

Email is inherently insecure -- it was designed without encryption in mind. Protecting your email communications requires layering multiple defenses.

### Step 1: Use an Encrypted Email Provider

ProtonMail and Tutanota encrypt your mailbox at rest. Messages between users of the same service are end-to-end encrypted automatically.

### Step 2: Use Email Aliases

Services like SimpleLogin and AnonAddy create unique forwarding addresses for each service you sign up for. When a service gets breached, you disable that single alias.

```bash
# Check if your email has been in known breaches
curl -s -H "hibp-api-key: YOUR_KEY" \
  "https://haveibeenpwned.com/api/v3/breachedaccount/your@email.com" | \
  python3 -m json.tool
```

### Step 3: Strip Metadata from Outgoing Emails

Email headers reveal your IP address, email client, and operating system. ProtonMail strips most headers automatically. For other providers, use Tor Browser to access webmail.

## Disk Encryption Setup

Full disk encryption protects your data if your device is lost or stolen. Every operating system now includes built-in encryption.

```bash
# Linux: Check if LUKS encryption is active
sudo cryptsetup status $(mount | grep ' / ' | cut -d' ' -f1)

# macOS: Check FileVault status
fdesetup status

# Verify encryption on an external drive
sudo cryptsetup luksDump /dev/sdb1
```

Enable encryption before storing sensitive data -- encrypting an existing drive is possible but riskier than starting fresh.

## Related Reading

- [Privacy Tools Troubleshooting Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
