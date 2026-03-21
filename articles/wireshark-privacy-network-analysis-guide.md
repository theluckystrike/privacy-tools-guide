---
layout: default
title: "Wireshark Basics for Privacy Network Analysis"
description: "How to use Wireshark to inspect network traffic from your own devices, identify unexpected connections, detect trackers, and verify encryption is working"
date: 2026-03-21
author: theluckystrike
permalink: /wireshark-privacy-network-analysis-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Wireshark is a graphical network protocol analyzer that lets you capture and inspect every packet flowing through your network interface. For privacy work, it answers questions that no other tool can: what is that app actually sending home, is my VPN leaking, which domains does my phone contact when I'm not touching it, and is this connection actually encrypted?

This guide covers the privacy-relevant use cases: finding unexpected connections, verifying encryption, and detecting data exfiltration.

## Installation

### Linux

```bash
# Debian/Ubuntu
sudo apt install wireshark

# During installation, select "yes" to allow non-root users to capture
# Then add your user to the wireshark group:
sudo usermod -aG wireshark $USER
# Log out and back in for group changes to take effect

# Fedora
sudo dnf install wireshark

# Arch
sudo pacman -S wireshark-qt
```

### macOS

```bash
brew install --cask wireshark
```

macOS requires granting Wireshark permission to capture packets. Wireshark will prompt you to install the ChmodBPF package on first launch.

### Windows

Download the official installer from wireshark.org. It includes Npcap (the Windows packet capture driver).

## Step 1: Capture Traffic on Your Interface

Launch Wireshark. The opening screen shows all available network interfaces and a live preview of traffic on each.

Select your active interface (usually:
- `eth0` or `enp3s0` for wired
- `wlan0` or `wlp2s0` for wireless
- `en0` on macOS)

Click the shark fin button or double-click the interface to start capturing.

**Stop the capture after 60 seconds** — even idle systems generate a lot of traffic. Save the capture file (File → Save As → `.pcapng`) before analyzing.

## Step 2: Filter for Interesting Traffic

The display filter bar at the top is where Wireshark's power lies. Filters narrow the captured packets to what you care about.

```
# All DNS queries — see what domains your system is looking up
dns

# All HTTP traffic (unencrypted — should be rare in 2026)
http

# All HTTPS/TLS traffic
tls

# Traffic to or from a specific IP
ip.addr == 93.184.216.34

# Traffic to or from a specific domain (requires DNS resolution)
# First find the IP via: host example.com
# Then filter by IP

# All outbound traffic from your machine (replace with your IP)
ip.src == 192.168.1.100

# All traffic except local network
ip.dst != 192.168.0.0/16 && ip.src != 192.168.0.0/16

# Packets with specific port
tcp.port == 443

# Unencrypted HTTP POST requests (potential data exfiltration)
http.request.method == "POST"
```

## Step 3: Identify Unexpected Connections

This is the core privacy use case. After capturing 60 seconds of idle traffic, filter for outbound connections to external IPs:

```
ip.src == YOUR_LOCAL_IP && !ip.dst == 192.168.0.0/16 && !ip.dst == 10.0.0.0/8
```

For each destination IP, look it up:

```bash
# Reverse DNS lookup
dig -x 93.184.216.34

# WHOIS to identify the organization
whois 93.184.216.34

# Or use a web service:
# https://ipinfo.io/93.184.216.34
```

Common unexpected connections to investigate:
- Connections to `*.amazonaws.com`, `*.google.com`, `*.microsoft.com` from apps that shouldn't need cloud services
- Connections immediately after startup before you've opened any apps (OS telemetry)
- Periodic connections every few minutes (background sync or telemetry beacons)

## Step 4: Verify VPN Tunnel Encryption

When connected to a VPN, all external traffic should appear encrypted and routed through the VPN endpoint.

Capture traffic while connected to a VPN, then apply this filter:

```
# Show all traffic NOT going to your VPN endpoint
# Replace 203.0.113.50 with your VPN server IP
ip.dst != 203.0.113.50 && ip.src != 203.0.113.50 && !ip.dst == 192.168.0.0/16
```

This filter should return **no results** if your VPN is working. Any external connections bypassing the VPN endpoint indicate a leak.

To check what protocol is protecting the VPN tunnel:

```
# WireGuard uses UDP
udp.port == 51820

# OpenVPN default
udp.port == 1194

# Check for plaintext content in captured packets
# Right-click any packet → Follow → TCP Stream
# Look for readable text in the stream content
```

## Step 5: Detect Unencrypted Data Transmission

Any packet transmitted over HTTP (not HTTPS) carries its content in plaintext. In Wireshark, follow the TCP stream of any HTTP connection to see exactly what was transmitted:

1. Filter: `http`
2. Right-click a packet → Follow → HTTP Stream
3. Red text is what your system sent; blue is the response

If you see readable personal data — usernames, form values, tracking identifiers — in HTTP traffic, that data is being transmitted without encryption.

To find POST requests with bodies (most likely to contain sensitive data):
```
http.request.method == "POST"
```

Follow the stream on each result to inspect the posted content.

## Step 6: Inspect TLS Certificates

For HTTPS connections, you can verify the certificate being used without decrypting the traffic:

Filter: `tls.handshake.type == 2` (ServerHello messages)

Right-click a packet → Follow → TLS Stream. Scroll to find the certificate details.

Or use the more targeted approach:

```
# Show all TLS Client Hello packets (your system initiating connections)
tls.handshake.type == 1

# Expand the packet details → Transport Layer Security → TLSv1.3 Record Layer
# → Handshake Protocol: Client Hello → Extension: server_name
# This shows the SNI (Server Name Indication) — the domain being connected to,
# visible even in encrypted TLS traffic
```

The SNI field reveals the hostname of every HTTPS connection, even though the content is encrypted. This is a known privacy limitation of TLS — only Encrypted Client Hello (ECH) hides the SNI.

## Step 7: Analyze a Mobile Device

To inspect traffic from a phone or other device on your network, you need to capture at the router level or use ARP spoofing to route the device's traffic through your capture machine.

The simpler approach: set up your laptop as a Wi-Fi hotspot that the phone connects to, then capture on the hotspot interface.

```bash
# Linux: create hotspot via nmcli
nmcli device wifi hotspot ifname wlan0 ssid CaptureHotspot password "testpass123"

# Start Wireshark on the hotspot interface
# (usually ap0 or a second wlan interface)
```

This routes all phone traffic through your laptop's wlan interface, which you can capture without any additional configuration on the phone.

## Using tshark for Command-Line Analysis

For automated analysis or remote servers without a GUI:

```bash
# Capture 60 seconds of traffic to file
sudo tshark -i eth0 -a duration:60 -w capture.pcapng

# Extract all unique destination IPs from a capture file
tshark -r capture.pcapng -T fields -e ip.dst | sort -u

# Extract all DNS queries
tshark -r capture.pcapng -Y dns -T fields -e dns.qry.name | sort -u

# Find HTTP POST requests
tshark -r capture.pcapng -Y "http.request.method == POST" -V | grep -A5 "Host:"
```

## Related Reading

- [How to Use Tcpdump to Verify VPN Traffic Is Encrypted](/a140-how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [VPN Kill Switch: How It Works and Which VPNs Have Real Ones](/vpn-kill-switch-how-it-works-which-vpns-have-real-ones/)
- [How to Detect Hidden Trackers in Android Apps](/detect-hidden-trackers-android-apps/)
- [Claude vs ChatGPT for Drafting Gdpr Compliant Privacy](https://theluckystrike.github.io/ai-tools-compared/claude-vs-chatgpt-for-drafting-gdpr-compliant-privacy-polici/)
- [Cursor Pro Privacy Mode Does It Cost Extra](https://theluckystrike.github.io/ai-tools-compared/cursor-pro-privacy-mode-does-it-cost-extra-for-zero-retention/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
