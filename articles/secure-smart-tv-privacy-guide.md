---
layout: default
title: "How to Secure Your Smart TV Privacy"
description: "Practical steps to disable ACR tracking, block smart TV telemetry at the network level, and reconfigure Samsung, LG, and Roku TVs for better privacy"
date: 2026-03-22
author: theluckystrike
permalink: /secure-smart-tv-privacy-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Secure Your Smart TV Privacy

Smart TVs are advertising terminals that also show video. Automatic Content Recognition (ACR) technology watches what you're watching and sends fingerprints to ad servers. Voice remotes with "always-on" mics are common. Telemetry runs constantly. This guide covers what to disable and how to block the rest at the network level.

---

## What Smart TVs Collect

**Automatic Content Recognition (ACR)**: Your TV captures screenshots of whatever is on screen — including HDMI input, streaming apps, and cable/antenna — and matches them against a database to identify what you're watching. This data is sold to advertisers.

**Voice data**: TVs with mic-enabled remotes record voice commands. Some older Samsung TVs had always-on mics; modern ones activate on button press, but the data still goes to cloud servers.

**Usage metrics**: App opens, viewing duration, button presses, and crashes are reported to the manufacturer.

**Ad identifiers**: Smart TVs have persistent advertising IDs that follow you across apps and streaming services.

The FTC fined Vizio $2.2M in 2017 for collecting viewing data without consent.

---

## Samsung Smart TV

### Disable ACR

Samsung calls their ACR system "Viewing Information Services":

```
Settings → Support → Terms & Privacy → Viewing Information Services → OFF

Also disable:
Settings → Support → Terms & Privacy → Interest-Based Advertisements → OFF
Settings → Support → Terms & Privacy → Voice Recognition Services → OFF
```

For older Samsung models (Tizen):
```
Menu → Smart Hub → Terms & Privacy → SyncPlus and Marketing → OFF
```

### Disable Personalized Ads

```
Settings → Support → Terms & Privacy → Personalized Ads → OFF
Settings → General → Privacy Choices → Personalized Ads Service → OFF
```

### Microphone

```
Settings → General → Voice → Bixby Voice Wake-up → OFF
```

---

## LG Smart TV (webOS)

### Disable ACR

LG calls it "Live Plus":

```
Settings → General → LivePlus → OFF
Settings → General → Additional Settings → Advertisement → Limit Ad Tracking → ON
Settings → General → Additional Settings → Advertisement → Reset Ad ID
```

### Disable AI Recommendation and Voice Data

```
Settings → General → AI Service → AI Recommendations → OFF
Settings → General → AI Service → Voice ID → OFF
Settings → All Settings → General → Privacy & Terms → User Agreements
→ Uncheck: Personalized Advertising Service, Voice Information
```

---

## Roku

Roku's business model is advertising, not hardware — aggressive data collection by default.

```
Settings → Privacy → Smart TV Experience → Use Info from TV Inputs → OFF
Settings → Privacy → Advertising → Limit Ad Tracking → ON
Settings → Privacy → Advertising → Reset Advertising Identifier
Settings → Privacy → Microphone → ON/OFF per app (revoke all unnecessary)
```

**Roku's limitation**: Even with all privacy settings disabled, Roku still collects device-level data per their privacy policy. Network-level blocking is required.

---

## Apple TV

```
Settings → Privacy → Analytics → Share Apple TV Analytics → OFF
Settings → Privacy → Tracking → Allow Apps to Request to Track → OFF
```

Apple TV doesn't have ACR. Individual apps (Netflix, Hulu) have their own tracking.

---

## Network-Level Blocking

Settings inside the TV OS can be changed by firmware updates. DNS blocking persists.

```bash
# Samsung telemetry domains (add to Pi-hole or /etc/hosts):
0.0.0.0 samsungads.com
0.0.0.0 log.samsung.com
0.0.0.0 ibs.samsung.com

# LG telemetry:
0.0.0.0 lgtvsdx.com
0.0.0.0 ibis.lgappstv.com

# Roku:
0.0.0.0 scribe.logs.roku.com
0.0.0.0 cooper.logs.roku.com
0.0.0.0 logs.roku.com

# Reload Pi-hole after editing custom list
pihole restartdns

# Watch what your TV queries in real-time
pihole -t | grep -i "samsung\|roku\|lgadv"
```

### Force TV Through Your DNS

Many smart TVs hard-code fallback DNS servers (Google's 8.8.8.8) to bypass router DNS:

```bash
# On OpenWrt: redirect all DNS queries from TV's IP to your Pi-hole
# Replace 192.168.1.55 with your TV's IP

uci add firewall redirect
uci set firewall.@redirect[-1].src=lan
uci set firewall.@redirect[-1].src_ip=192.168.1.55
uci set firewall.@redirect[-1].dest_port=53
uci set firewall.@redirect[-1].target=DNAT
uci set firewall.@redirect[-1].dest_ip=192.168.1.10   # Pi-hole IP
uci commit firewall && /etc/init.d/firewall restart
```

---

## Isolate TV on Its Own VLAN

```bash
# Create a separate VLAN for the TV with no access to your main LAN
# See: Network Segmentation for IoT Devices guide for full VLAN setup
```

---

## What Can't Be Blocked

Even with aggressive network blocking, some features stop working:
- Software updates (whitelist update servers)
- Some streaming app functionality
- Cloud-dependent features (voice assistants, smart home integration)

The cleanest solution: use a "dumb TV" (older set or commercial display without smart TV OS) connected to a streaming device you control separately. Your TV becomes a monitor.

---

## Related Reading

- [How to Tell if Your Smart TV Is Spying on You](/privacy-tools-guide/how-to-tell-if-your-smart-tv-is-spying-on-you/)
- [Smart TV Tracking What Data Samsung LG Vizio Collect](/privacy-tools-guide/smart-tv-tracking-what-data-samsung-lg-vizio-collect-about-v/)
- [Network Segmentation for IoT Devices](/privacy-tools-guide/iot-network-segmentation-vlan-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
