---

layout: default
title: "Threat Model for Undocumented Immigrant Protecting Location and Identity Digital"
description: "A practical guide to building a digital threat model for protecting location and identity. Learn actionable strategies for device hardening, network security, and operational security for at-risk individuals."
date: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-undocumented-immigrant-protecting-location-/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
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

Protecting location and identity requires layering multiple defenses. No single measure is sufficient, but each layer makes comprehensive compromise more difficult. Start with the fundamentals—device hardening and network protection—then add communication security and identity separation as your threat model requires.

The goal is not perfect security, which is unattainable. The goal is making targeted surveillance costly enough that adversaries choose easier targets. Every layer of protection raises the resource requirements for those seeking to track you, creating meaningful distance between you and potential threats to your safety.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
