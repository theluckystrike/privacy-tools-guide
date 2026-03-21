---
layout: default
title: "Guides — Complete Guide"
description: "Browse every privacy and security guide in one place — from VPN setup and password manager reviews to encrypted messaging comparisons and browser"
date: 2026-03-15
last_modified_at: 2026-03-15
permalink: /guides-hub/
categories: [guides]
reviewed: true
score: 7
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

## Related Reading

- [Privacy Tools Troubleshooting Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
