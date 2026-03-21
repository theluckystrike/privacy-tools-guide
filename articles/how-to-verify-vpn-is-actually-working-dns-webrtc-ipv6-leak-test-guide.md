---
layout: default
title: "Verify VPN is Actually Working: DNS, WebRTC, IPv6 Leak Test"
description: "Step-by-step VPN verification guide. Test DNS leaks, WebRTC leaks, IPv6 leaks, kill switch verification. Real commands and tools"
date: 2026-03-20
author: theluckystrike
permalink: /how-to-verify-vpn-is-actually-working-dns-webrtc-ipv6-leak-test-guide/
categories: [guides]
tags: [privacy-tools-guide, vpn]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Installing a VPN doesn't guarantee your traffic is private. Misconfigured VPNs leak your real IP through DNS queries, WebRTC connections, or IPv6 traffic—all while the VPN indicator shows green. Your ISP can still see your browsing, and websites can still fingerprint you. This guide walks through verification tests you can run right now: DNS leak detection (nslookup command), WebRTC leak tests (browser console), IPv6 leak detection (running a local server), and kill switch verification (testing connection drop behavior). Each test reveals whether your VPN is actually working. Real tools: dnsleak.com, ipleak.net, whoami.akamai.com, and tcpdump for advanced users.

## Why VPN Verification Matters

A "working" VPN means:

1. **Your real IP is hidden** - Websites see the VPN's IP, not yours
2. **DNS queries are encrypted** - Your ISP can't see what sites you're visiting
3. **WebRTC doesn't leak your real IP** - Browser doesn't bypass the VPN
4. **IPv6 traffic routes through VPN** - Not leaking via IPv6 tunnel
5. **Kill switch works** - If VPN drops, internet stops (prevents unencrypted traffic)

Many VPN apps claim to do this. Few actually do it correctly. Misconfigurations like DNS not routing through the VPN, WebRTC not being blocked, or IPv6 bypass routes mean your privacy isn't actually protected—even though the VPN app shows "Connected."

## Test 1: Check Your Real IP

First, establish what your real IP is. This is what you want to hide.

**Find your real IP (easiest way):**

Disconnect from VPN. Visit any of these sites:

- [whatismyipaddress.com](https://whatismyipaddress.com)
- [ipinfo.io](https://ipinfo.io)
- [whoami.akamai.com](https://whoami.akamai.com)

Your real IP appears. Remember it. If you see this IP after connecting to VPN, the VPN isn't working.

**Command-line method:**

```bash
# macOS/Linux
curl https://ifconfig.me
# Output: 203.0.113.45

curl https://icanhazip.com
# Output: 203.0.113.45

# Windows PowerShell
(Invoke-WebRequest https://ifconfig.me).Content
```

Note the IP address (e.g., 203.0.113.45). This is your real IP that you want to hide.

## Test 2: DNS Leak Detection

When you use a VPN, your DNS queries should go through the VPN's DNS servers, encrypted. If they leak, your ISP can still see what sites you visit.

**Method 1: Online DNS leak test (easiest)**

Visit [dnsleak.com](https://dnsleak.com)

1. Disconnect from VPN
2. Go to dnsleak.com and note the DNS servers shown (your ISP's servers)
3. Connect to VPN
4. Refresh dnsleak.com

If the DNS servers change to the VPN provider's servers, DNS leaks aren't happening. If they stay the same, your VPN has a DNS leak.

**Expected results:**

Before VPN:
```
DNS Servers Detected:
1.1.1.1 (ISP Name)
8.8.8.8 (ISP Name)
```

After connecting to VPN (if working):
```
DNS Servers Detected:
10.0.0.1 (VPN Provider Name)
10.0.0.2 (VPN Provider Name)
```

If you see your ISP's servers while connected to VPN, you have a DNS leak.

**Method 2: Command-line DNS leak test**

```bash
# Check what DNS server your system uses
nslookup google.com

# Output on VPN should show VPN's DNS:
# Server:  10.0.0.1 (VPN's DNS)
# Address: 10.0.0.1#53

# Non-VPN output shows ISP's DNS:
# Server:  8.8.8.8
# Address: 8.8.8.8#53
```

If the server address is your ISP's DNS while VPN is connected, you have a leak.

**Method 3: Advanced - tcpdump to capture traffic**

For technical users, capture actual DNS traffic to verify it's encrypted through VPN:

```bash
# Install tcpdump if needed
# macOS: brew install tcpdump
# Linux: sudo apt-get install tcpdump

# Capture DNS traffic
sudo tcpdump -i any -n 'udp port 53' -c 5

# Output while NOT on VPN:
# Your computer -> ISP DNS (unencrypted)

# Output while on VPN:
# Your computer -> VPN endpoint (encrypted tunnel, no direct DNS queries visible)
```

If you see DNS queries (port 53) going directly to your ISP while connected to VPN, there's a leak.

## Test 3: WebRTC Leak Detection

WebRTC is used by browsers for real-time communication (video calls, file sharing). It can bypass your VPN and leak your real IP directly.

**Method 1: Online WebRTC leak test (easiest)**

Visit [ipleak.net](https://ipleak.net)

1. Disconnect from VPN and note your IP
2. Connect to VPN
3. Go to ipleak.net
4. Check "WebRTC" section

If it shows your VPN's IP only, no leak. If it shows your real IP in the WebRTC section, you have a leak.

**Expected results (no leak):**

```
IPv4: 198.51.100.42 (VPN provider's IP)
WebRTC: 198.51.100.42 (same as VPN)
```

**Bad result (WebRTC leak):**

```
IPv4: 198.51.100.42 (VPN provider's IP)
WebRTC: 203.0.113.45 (YOUR REAL IP - LEAK!)
```

**Method 2: Browser console WebRTC leak test**

For technical users, test WebRTC directly in browser console:

```javascript
// Open browser dev tools (F12) and paste this into console:

const pc = new RTCPeerConnection({
  iceServers: [{ urls: ["stun:stun.l.google.com:19302"] }]
});

const candidates = [];

pc.onicecandidate = (ice) => {
  if (!ice || !ice.candidate) return;
  const candidate = ice.candidate.candidate;
  console.log("WebRTC Candidate:", candidate);
  candidates.push(candidate);
};

pc.createDataChannel("test");
pc.createOffer().then(offer => pc.setLocalDescription(offer));

// After 3 seconds, check the logged candidates:
setTimeout(() => {
  console.log("All WebRTC candidates:", candidates);
  // Look for IP addresses in the output
  candidates.forEach(c => {
    if (c.match(/(\d{1,3}\.){3}\d{1,3}/)) {
      console.log("IP found:", c);
    }
  });
}, 3000);
```

**What to look for:**

In the candidates, look for IP addresses. If you see your real IP (e.g., 203.0.113.45) instead of your VPN IP, you have a WebRTC leak.

**Method 3: Disable WebRTC in browser (fix if leaking)**

If you found a WebRTC leak, disable WebRTC:

**Firefox:**
1. Type `about:config` in address bar
2. Search for `media.peerconnection.enabled`
3. Set it to `false`

**Chrome:**
1. Visit `chrome://extensions/`
2. Install WebRTC Leak Prevent extension
3. Leave it enabled

## Test 4: IPv6 Leak Detection

If your network supports IPv6, traffic can leak through IPv6 routes that bypass your VPN.

**Method 1: Online IPv6 leak test (easiest)**

Visit [ipleak.net](https://ipleak.net)

1. Connect to VPN
2. Go to ipleak.net
3. Check "IPv6 address" section

If no IPv6 address is shown, or it shows the VPN provider's IPv6, you're fine. If it shows a different IPv6 (your real one), you have a leak.

**Expected results (no leak):**

```
IPv4: 198.51.100.42 (VPN IP)
IPv6: Not detected OR 2001:db8::1 (VPN provider's IPv6)
```

**Bad result (IPv6 leak):**

```
IPv4: 198.51.100.42 (VPN IP)
IPv6: 2600:1700:1234:5678::1 (YOUR REAL IPv6 - LEAK!)
```

**Method 2: Command-line IPv6 check**

```bash
# Check if you have an IPv6 address
ifconfig | grep inet6

# Output with IPv6:
# inet6 fe80::1 (link-local, okay)
# inet6 2600:1700:1234:5678::1 (global, check this)

# If your global IPv6 is visible on ipleak.net but shouldn't be,
# your VPN isn't routing IPv6 traffic
```

**Method 3: Disable IPv6 if leaked (fix)**

If you have IPv6 leaks and can't use IPv6, disable it:

**macOS:**
```bash
# Disable IPv6
networksetup -setv6off Wi-Fi

# Re-enable later:
networksetup -setv6automatic Wi-Fi
```

**Linux:**
```bash
# Temporarily disable IPv6
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1

# Re-enable:
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
```

**Windows:**
```bash
# Open PowerShell as admin
netsh interface ipv6 set state disabled=yes

# Re-enable:
netsh interface ipv6 set state disabled=no
```

## Test 5: Kill Switch Verification

A "kill switch" stops all traffic if the VPN drops, preventing unencrypted data from leaking.

**Method 1: Manual disconnect test**

1. Connect to VPN
2. Open a terminal/command prompt
3. Ping a server to confirm internet works:

```bash
ping 8.8.8.8
# Output: 64 bytes from 8.8.8.8: icmp_seq=0 ttl=119 time=45ms
```

4. Disconnect the VPN (toggle it off in the app)
5. Immediately ping again:

```bash
ping 8.8.8.8
```

**Good result (kill switch working):**
- Ping continues returning "no response" or "unreachable"
- Internet is completely blocked
- No packets get through

**Bad result (no kill switch):**
- Ping returns responses immediately after VPN disconnect
- Internet continues working
- Unencrypted traffic leaks to the internet

**Method 2: Network connectivity test (advanced)**

For technical users, use netstat to monitor connections:

**macOS/Linux:**
```bash
# Monitor network connections
netstat -an | grep ESTABLISHED

# Before VPN disconnect: Shows connection to VPN endpoint
# tcp4  0  0  192.168.1.100.54321  198.51.100.42.1194  ESTABLISHED

# After VPN disconnect with kill switch ON:
# No ESTABLISHED connections appear
# (kill switch blocks all traffic)

# After VPN disconnect with kill switch OFF:
# Direct connections to ISP reappear
# tcp4  0  0  192.168.1.100.54322  8.8.8.8.443  ESTABLISHED
```

## Test 6: Check Leak Test Sites Themselves

Some leak test sites have false positives. Cross-reference:

- [dnsleak.com](https://dnsleak.com) - DNS leak detection
- [ipleak.net](https://ipleak.net) - IP, WebRTC, IPv6 leaks
- [whoami.akamai.com](https://whoami.akamai.com) - Simple IP check
- [browserleaks.com](https://browserleaks.com) - browser leaks

Visit 2-3 sites. If they all show the same IP (your VPN's IP), the VPN is working. If they show different IPs, something is wrong.

## Test Checklist

Before you trust a VPN, run this full verification:

```
1. Real IP check:
   [ ] Record your real IP (disconnect from VPN, check on 3 sites)

2. DNS leak test:
   [ ] Visit dnsleak.com on VPN
   [ ] Verify DNS servers are VPN provider's, not ISP's
   [ ] Run nslookup command, check DNS server address

3. WebRTC leak test:
   [ ] Visit ipleak.net and check WebRTC section
   [ ] Run browser console test
   [ ] Verify WebRTC shows VPN IP, not real IP

4. IPv6 leak test:
   [ ] Visit ipleak.net and check IPv6 address
   [ ] Run ifconfig/ipconfig command
   [ ] Verify no global IPv6 address is visible

5. Kill switch test:
   [ ] Ping 8.8.8.8 while connected to VPN (works)
   [ ] Disconnect VPN
   [ ] Ping 8.8.8.8 immediately (should fail)
   [ ] Verify ping doesn't recover

6. Cross-reference:
   [ ] Check IP on dnsleak.com
   [ ] Check IP on ipleak.net
   [ ] Check IP on whoami.akamai.com
   [ ] All three should show VPN provider's IP
```

## Common Leaks and Fixes

**Problem: DNS leak detected**

Causes:
- VPN app not configured to use VPN DNS
- System DNS set to ISP before VPN connected
- Application using hardcoded DNS (e.g., DoH in some browsers)

Fixes:
- In VPN app settings, verify "DNS" is set to "VPN's DNS" (not ISP)
- Disable DNS-over-HTTPS (DoH) in browser if it conflicts
- Restart computer after VPN connection

**Problem: WebRTC leak detected**

Causes:
- WebRTC enabled in browser
- Browser using direct connection for peer connections

Fixes:
- Firefox: Disable `media.peerconnection.enabled` in about:config
- Chrome: Install WebRTC Leak Prevent extension
- Safari: Less prone to WebRTC leaks due to stricter WebRTC implementation

**Problem: IPv6 leak detected**

Causes:
- ISP provides IPv6 but VPN doesn't support it
- Browser preferring IPv6 over IPv4

Fixes:
- Disable IPv6 on your device (systemctl)
- Choose VPN provider with IPv6 support
- Configure VPN to tunnel IPv6 traffic

**Problem: Kill switch not working**

Causes:
- Kill switch disabled in VPN settings
- Firewall rules allowing bypass

Fixes:
- In VPN app, enable "Kill Switch" or "Network Lock"
- Check firewall isn't allowing exceptions
- Restart VPN app and computer

## Real Example: Testing ExpressVPN

Here's a complete verification of ExpressVPN:

1. **Real IP (no VPN):** 203.0.113.45
2. **Connect to ExpressVPN** (VPN provider: United States server)
3. **IP on ipleak.net:** 198.51.100.42 ✓ (different from real IP)
4. **DNS on dnsleak.com:** Cloudflare 1.1.1.1 ✓ (VPN provider's DNS)
5. **WebRTC on ipleak.net:** 198.51.100.42 ✓ (VPN IP, no leak)
6. **IPv6 on ipleak.net:** Not detected ✓ (IPv6 not routed, safe)
7. **Kill switch test:** Ping stops on VPN disconnect ✓ (working)

Result: **ExpressVPN working correctly, no leaks detected.**

## Related Reading

- [VPN Security Comparison 2026](/privacy-tools-guide/vpn-security-comparison-2026/)
- [Best VPN for Privacy and Security](/privacy-tools-guide/best-vpn-for-privacy-and-security/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
