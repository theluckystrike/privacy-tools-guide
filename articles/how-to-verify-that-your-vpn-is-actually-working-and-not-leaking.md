---
layout: default
title: "Verify That Your VPN Is Actually Working and Not Leaking"
description: "Learn how to verify your VPN is genuinely protecting your traffic. Test methods for DNS leaks, IP leaks, WebRTC leaks, and confirm encryption is working."
date: 2026-03-20
author: theluckystrike
permalink: /how-to-verify-that-your-vpn-is-actually-working-and-not-leaking/
categories: [guides, security]
tags: [privacy-tools-guide, security, vpn]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

Installing a VPN creates a false sense of security if you don't verify it's actually functioning. A misconfigured VPN client can leak your real IP address, DNS queries, or WebRTC connections while appearing connected. This guide shows how to test your VPN systematically using command-line tools and online services, ensuring your traffic is genuinely encrypted and routed through your VPN provider's servers, not accidentally exposed to your ISP or network observer.

## What VPN Leaks Actually Mean

A VPN leak occurs when traffic destined for encrypted tunneling bypasses the tunnel and travels on your regular network connection. Three main leak types affect different layers:

**DNS Leaks:** Your DNS queries (website lookups) bypass the VPN and go to your ISP's DNS servers, revealing what sites you're visiting even though HTTP traffic is encrypted.

**IP Leaks:** Your real public IP address gets exposed in HTTP headers, WebRTC connections, or other protocols, defeating the anonymity purpose of a VPN.

**WebRTC Leaks:** Real-time communication protocols used by browsers can leak your local and real IP addresses even through a VPN connection.

Detecting these requires testing at each layer of the network stack, which is why a single online test isn't sufficient for verification.

## Step 1: Verify Your Public IP Has Changed

This is the most basic test. Before connecting to VPN, note your real IP address:

```bash
# Check your current public IP
curl https://icanhazip.com
# Output: 203.0.113.42 (your real IP)

# Alternative methods
curl https://ipinfo.io
# Returns: {"ip":"203.0.113.42","city":"San Francisco",...}

curl https://api.ipify.org
# Output: 203.0.113.42
```

After connecting to your VPN, run the same command:

```bash
# After VPN connection
curl https://icanhazip.com
# Output: 198.51.100.50 (VPN server IP)
```

If the IP is identical before and after VPN connection, your VPN client is not routing traffic through the tunnel. This indicates a complete connection failure.

## Step 2: Test for DNS Leaks

DNS leaks are the most common and often go undetected. Your traffic might be encrypted, but DNS queries reveal which websites you're visiting.

### Method 1: Using dnsleaktest.com

Visit [https://www.dnsleaktest.com/](https://www.dnsleaktest.com/) while connected to your VPN and click "Extended test." Results show which DNS servers are handling your queries.

**Expected result:** DNS servers should belong to your VPN provider (e.g., "ProtonVPN DNS," "ExpressVPN DNS"), not your ISP.

**Failure case:** If you see your ISP's DNS servers (e.g., "Comcast DNS," "Verizon DNS"), your DNS queries are leaking.

### Method 2: CLI-based DNS Testing

For programmatic verification:

```bash
# Install necessary tools
# macOS
brew install bind-tools

# Linux
sudo apt install dnsutils

# Test DNS resolution
nslookup google.com 8.8.8.8
# Run from terminal while connected to VPN

# Expected output with VPN:
# Server: [VPN DNS server, e.g., 10.8.0.1]
# Address: [VPN DNS server address]

# Failure: If "Server" shows your ISP's server, DNS is leaking
```

### Method 3: Compare DNS Servers Before and After

```bash
# Before VPN connection
cat /etc/resolv.conf
# Shows your ISP's DNS servers

# After VPN connection
cat /etc/resolv.conf
# Should show VPN provider's DNS servers
# If identical to before, DNS is misconfigured
```

## Step 3: Test for WebRTC Leaks

WebRTC (Web Real-Time Communication) used for video/audio calls can leak your real IP address. Many browsers expose local network IPs through WebRTC regardless of VPN status.

### Method 1: Online WebRTC Leak Test

Visit [https://browserleaks.com/webrtc](https://browserleaks.com/webrtc) while connected to your VPN.

**Expected result:** "No WebRTC leaks detected" or only shows VPN IP addresses in the results.

**Failure case:** Shows your real public IP or local private IP (192.168.x.x, 10.0.0.x).

### Method 2: JavaScript Console Test

Open browser developer console (F12) and run:

```javascript
function findIP(onNewIP) {
  var myPeerConnection = window.RTCPeerConnection || window.webkitRTCPeerConnection;
  if (!myPeerConnection) {
    onNewIP('WebRTC not supported');
    return;
  }
  var pc = new myPeerConnection({iceServers:[]}),
    ips = [],
    mediaStream = new Promise(resolve => navigator.mediaDevices.getUserMedia({audio:false,video:{width:640}}).then(resolve)),
    onIce = function(ice) {
      if (ice.candidate.candidate.match(/candidate:(.*) /)[1];
      var ip_regex = /([0-9]{1,3}(\.[0-9]{1,3}){3})/
      var ipaddr = ip_regex.exec(ice.candidate.candidate)[1];
      if (ipaddr != '127.0.0.1' && ips.indexOf(ipaddr) == '-1') ips.push(ipaddr);
    };
  pc.onicecandidate = onIce;
  pc.createDataChannel('');
  pc.createOffer().then(offer => pc.setLocalDescription(offer));
  mediaStream.then(stream => stream.getTracks().forEach(track => pc.addTrack(track, stream)));
  setTimeout(() => { onNewIP(ips.length > 0 ? ips.join(',') : 'No IP found'); }, 3000);
}
findIP(ip => console.log('Your IPs:', ip));
```

**Expected result:** Only shows IPs belonging to your VPN provider.

**Failure result:** Shows 192.168.x.x or 10.0.0.x (local network) or your real public IP.

## Step 4: Check Encryption Status

Verify that traffic is actually encrypted by inspecting packet headers.

### Using tcpdump (macOS/Linux)

```bash
# Capture traffic while connected to VPN
sudo tcpdump -i any -n -A port 443
# Look for encrypted (unreadable) traffic

# Example output with working VPN:
# (shows encrypted TLS handshake, not readable HTTP)
# ...encrypted bytes... .w.....
# HTTP content should NOT appear in plaintext

# Example output without VPN:
# Can see plaintext HTTP headers like "GET /search?q=..."
# This indicates unencrypted traffic
```

### Disable VPN Temporarily and Compare

```bash
# 1. Disconnect from VPN
# 2. Run tcpdump again
# 3. Visit a non-HTTPS website (http://example.com)
# 4. Look for plaintext HTTP headers

# With VPN:
# All packets encrypted, no readable content

# Without VPN:
# Plaintext HTTP visible:
# GET /page HTTP/1.1
# Host: example.com
```

## Comprehensive VPN Verification Checklist

```bash
#!/bin/bash
# VPN Verification Script

echo "=== VPN VERIFICATION TEST ==="
echo ""

echo "1. PUBLIC IP ADDRESS"
echo "Before VPN (should show ISP IP):"
curl -s https://icanhazip.com
echo "Connect to VPN, then press Enter..."
read

echo "After VPN (should show different IP):"
AFTER_IP=$(curl -s https://icanhazip.com)
echo $AFTER_IP
echo ""

echo "2. DNS LEAK TEST"
echo "Connected to VPN DNS servers:"
nslookup google.com | grep -A1 "Server:"
echo ""

echo "3. GEOLOCATION TEST"
curl -s https://ipinfo.io | jq '.city, .country'
echo ""

echo "4. CERTIFICATE VERIFICATION"
echo "Check certificate chain (should be encrypted):"
openssl s_client -connect google.com:443 2>/dev/null | grep -A5 "subject="
```

Save as `vpn-verify.sh`, make executable with `chmod +x vpn-verify.sh`, and run:

```bash
./vpn-verify.sh
```

## VPN Configuration Best Practices

**Always verify DNS configuration in VPN settings:**

```
Most VPN clients have a "DNS settings" section. Ensure:
- Primary DNS: VPN provider's DNS (not 8.8.8.8 or 1.1.1.1)
- Secondary DNS: VPN provider's backup DNS
- Never use ISP DNS while connected to VPN
```

**Enable kill switch if available:**

```
VPN kill switch = disconnect internet if VPN drops
This prevents your real IP from leaking during temporary connection loss
```

**Disable IPv6 if only IPv6 is available:**

Some VPN clients only tunnel IPv4 traffic, leaving IPv6 exposed. Check in privacy settings.

## Comparison Table: VPN Testing Methods

| Test | Method | Difficulty | Reliability | Time |
|------|--------|-----------|------------|------|
| Public IP change | Command line | Easy | Excellent | 1 min |
| DNS leak | dnsleaktest.com | Easy | Excellent | 2 min |
| WebRTC leak | browserleaks.com | Easy | Good | 2 min |
| Encryption verification | tcpdump + analysis | Hard | Excellent | 10 min |
| Full audit | Script (all tests) | Medium | Excellent | 15 min |

## Related Reading

- [VPN Leak Testing Guide - IVPN](https://www.ivpn.net/knowledgebase/article/45-will-my-isp-know-that-i-am-using-a-vpn)
- [Wireshark Network Analysis Guide](https://www.wireshark.org/docs/)
- [RFC 8446: TLS 1.3 Specification](https://tools.ietf.org/html/rfc8446)

## Footer

Regular VPN verification is essential security hygiene. Test after major software updates, when switching VPN providers, and quarterly as a security audit. Don't trust VPN marketing claims that promise "bank-grade encryption"—verify directly using the methods in this guide. A VPN providing false security is worse than no VPN, as it creates unwarranted confidence in protection that isn't actually occurring. Test methodically, document results, and only trust VPNs that pass all four verification layers.

{% endraw %}
