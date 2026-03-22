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

**Automatic Content Recognition (ACR)**: Your TV captures screenshots of whatever is on screen — including HDMI input, streaming apps, and cable/antenna — and matches them against a database to identify what you're watching. This data is sold to advertisers and data brokers. ACR operates regardless of what you're watching; it sees your cable box, your gaming console, and your Blu-ray player just as well as it sees built-in apps.

**Voice data**: TVs with mic-enabled remotes record voice commands. Some older Samsung TVs had always-on mics; modern ones activate on button press, but the data still goes to cloud servers for processing. The voice data is used to improve voice recognition models, and may be stored for months.

**Usage metrics**: App opens, viewing duration, button presses, and crashes are reported to the manufacturer. This includes timestamps of when you turn the TV on and off, which apps you use and for how long, and what buttons you press on the remote.

**Ad identifiers**: Smart TVs have persistent advertising IDs that follow you across apps and streaming services, similar to the advertising ID on a smartphone. These IDs are shared with third-party advertisers embedded in every streaming app.

The FTC fined Vizio $2.2M in 2017 for collecting viewing data without consent. Vizio was capturing ACR data and selling it to data brokers without adequately disclosing this to users. Samsung, LG, and Roku have all faced similar regulatory scrutiny in various jurisdictions.

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

Also navigate to `Settings → General → System Manager → Smart Security` and disable it if you don't use Samsung's built-in security features — this service collects app usage data.

### Microphone

```
Settings → General → Voice → Bixby Voice Wake-up → OFF
```

On models with a remote that has a physical mic button, the mic should only activate when the button is held — but disabling Bixby wake-up ensures it is not listening for a wake word at other times. If your remote has a mic button you never use, a piece of tape over the microphone hole on the remote is a reliable physical control.

### Samsung IoT Integration

If you use Samsung SmartThings or have connected your TV to a Samsung account, you should also review:

```
Settings → General → External Device Manager → Device Connect Manager
→ Disable "Device Share" if not needed
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

On newer webOS versions (webOS 22 and later), LG has reorganized these menus. Navigate to `All Settings → Support → Privacy Notice` and look through each category for data collection toggles. LG bundles multiple settings under ambiguous headings, so read each one carefully rather than clicking through.

### LG ThinQ Account

If you signed into an LG account on your TV, that account links your viewing data to your identity across LG devices. Consider using the TV without an LG account — most features work without one. To sign out: `Settings → General → LG Account → Sign Out`.

---

## Roku

Roku's business model is advertising, not hardware — aggressive data collection by default.

```
Settings → Privacy → Smart TV Experience → Use Info from TV Inputs → OFF
Settings → Privacy → Advertising → Limit Ad Tracking → ON
Settings → Privacy → Advertising → Reset Advertising Identifier
Settings → Privacy → Microphone → ON/OFF per app (revoke all unnecessary)
```

**Roku's limitation**: Even with all privacy settings disabled, Roku still collects device-level data per their privacy policy. Their 2023 privacy policy explicitly states they share "inferred data" with advertising partners even when ad tracking is limited. Network-level blocking is required for meaningful privacy.

Roku also has a channel data sharing setting that is separate from the advertising settings:

```
Settings → Privacy → Channel Data Sharing → OFF
```

This controls whether Roku shares your channel usage data with third parties for market research.

---

## Apple TV

```
Settings → Privacy → Analytics → Share Apple TV Analytics → OFF
Settings → Privacy → Tracking → Allow Apps to Request to Track → OFF
```

Apple TV doesn't have ACR built into the OS. Individual apps (Netflix, Hulu, YouTube) have their own tracking that operates independently of Apple's settings. The App Tracking Transparency prompt that appears when you first open each app controls whether that app can share your data with third-party advertisers — denying all of these is the right default.

Apple's own apps collect usage data regardless of these settings when you're signed into an Apple ID. If you want to minimize Apple's data collection, use Apple TV without signing into an Apple ID for the TV itself (though Apple TV+ requires one).

---

## Network-Level Blocking

Settings inside the TV OS can be changed by firmware updates — manufacturers have pushed updates that quietly re-enabled tracking. DNS blocking at the router persists regardless of firmware changes.

```bash
# Samsung telemetry domains (add to Pi-hole or /etc/hosts):
0.0.0.0 samsungads.com
0.0.0.0 log.samsung.com
0.0.0.0 ibs.samsung.com
0.0.0.0 cdn.samsungcloud.com
0.0.0.0 samsungacr.com

# LG telemetry:
0.0.0.0 lgtvsdx.com
0.0.0.0 ibis.lgappstv.com
0.0.0.0 lgsmartad.com
0.0.0.0 smartshare.lgtvsdx.com

# Roku:
0.0.0.0 scribe.logs.roku.com
0.0.0.0 cooper.logs.roku.com
0.0.0.0 logs.roku.com
0.0.0.0 auction.roku.com

# Reload Pi-hole after editing custom list
pihole restartdns

# Watch what your TV queries in real-time
pihole -t | grep -i "samsung\|roku\|lgadv"
```

Pi-hole v5 and later lets you create group-based blocking rules. Create a "smart-tv" group and assign your TV's IP to it, then apply a stricter blocklist to that group only. This avoids accidentally breaking other devices that might legitimately query some of these domains.

### Force TV Through Your DNS

Many smart TVs hard-code fallback DNS servers (Google's 8.8.8.8) to bypass router DNS. Samsung and Roku are particularly known for this. Redirect all DNS queries from the TV's IP to Pi-hole:

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

On pfSense/OPNsense, create a NAT redirect rule under Firewall → NAT → Port Forward that redirects TCP/UDP port 53 from the TV's IP to your Pi-hole. This catches DoH (DNS-over-HTTPS) only if you also block port 443 to known DoH endpoints — but most smart TVs use plain DNS, not DoH.

### Monitor Your TV's Traffic

Before blocking, observe what domains your TV queries during normal use:

```bash
# On Pi-hole, filter query log by TV's IP:
# Admin panel → Query Log → filter by client IP

# To capture raw traffic with tcpdump (on router or separate capture device):
tcpdump -i br-lan -n host 192.168.1.55 -w /tmp/tv-capture.pcap

# Analyze with tshark:
tshark -r /tmp/tv-capture.pcap -T fields -e dns.qry.name | sort -u
```

This tells you exactly what your TV is contacting and lets you make informed blocklist decisions.

---

## Isolate TV on Its Own VLAN

Network isolation prevents your smart TV from communicating with other devices on your local network — useful if you're concerned about lateral movement in the event the TV OS is compromised, or simply want to limit what the TV can reach.

```bash
# Create a separate VLAN for the TV with no access to your main LAN
# Block inter-VLAN routing from TV VLAN to your main LAN VLAN
# Allow TV VLAN outbound to internet (for streaming) on whitelisted ports only
# See: Network Segmentation for IoT Devices guide for full VLAN setup
```

With VLAN isolation, your TV can stream Netflix but cannot scan or communicate with your NAS, desktop, or other LAN devices. Combined with DNS blocking, this is the strongest practical configuration short of using a dumb TV.

---

## What Can't Be Blocked

Even with aggressive network blocking, some features stop working:

- Software updates (whitelist update servers or accept periodic unblocking for updates)
- Some streaming app functionality (Roku in particular requires its logging endpoints for normal function)
- Cloud-dependent features (voice assistants, smart home integration)

The cleanest solution: use a "dumb TV" (older set or commercial display without smart TV OS) connected to a streaming device you control separately. A commercial display — Philips, NEC, or Samsung's commercial monitors — has no smart TV OS, no ACR, and no telemetry. Connect it to an Apple TV, NVIDIA Shield, or a Raspberry Pi running Kodi for a privacy-respecting smart TV stack you control end-to-end.

---

## Related Reading

- [How to Tell if Your Smart TV Is Spying on You](/privacy-tools-guide/how-to-tell-if-your-smart-tv-is-spying-on-you/)
- [Smart TV Tracking What Data Samsung LG Vizio Collect](/privacy-tools-guide/smart-tv-tracking-what-data-samsung-lg-vizio-collect-about-v/)
- [Network Segmentation for IoT Devices](/privacy-tools-guide/iot-network-segmentation-vlan-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
