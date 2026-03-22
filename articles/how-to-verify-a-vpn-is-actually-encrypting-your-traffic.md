---
layout: default
title: "How to Verify a VPN Is Actually Encrypting Your Traffic"
description: "Practical guide to verifying your VPN is actually encrypting traffic. Covers Wireshark inspection, DNS leak tests, kill switch verification, and WebRTC"
date: 2026-03-21
last_modified_at: 2026-03-21
author: "Privacy Tools Guide"
permalink: /how-to-verify-a-vpn-is-actually-encrypting-your-traffic/
categories: [guides]
tags: [privacy-tools-guide, tools, vpn]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

You cannot trust a VPN's marketing claims—you must verify encryption yourself. Many VPN services claim to encrypt traffic while leaking DNS requests, exposing your real IP via WebRTC, or failing kill switches. This guide covers practical tests you can run with free tools to verify your VPN actually protects you: packet inspection with Wireshark, DNS leak detection, kill switch verification, WebRTC leak detection, and IPv6 leak checks. These aren't theoretical—they catch real failures.

## Table of Contents

- [Why Verification Matters](#why-verification-matters)
- [Prerequisites](#prerequisites)
- [Troubleshooting](#troubleshooting)
- [Related Reading](#related-reading)

## Why Verification Matters

A VPN's job is simple: encrypt your traffic so your ISP, network administrator, or monitoring service cannot see what you're doing. The promise is confidentiality through encryption.

In practice, many VPNs leak your real IP address through other protocols (DNS, WebRTC, IPv6). Others claim to encrypt but actually don't, or have kill switches that don't actually kill. Worse, some are honeypots—services run by security agencies to collect information.

You cannot verify the inner workings of a commercial VPN. You can only verify observable behavior. These tests show you whether your VPN is actually doing what it claims.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: What Encryption Looks Like: Baseline Testing

Before testing your VPN, establish what encrypted and unencrypted traffic looks like.

### Unencrypted (No VPN) - See All Your Traffic

First, observe your real traffic without a VPN. Open Wireshark and capture packets while visiting a plain HTTP website (not HTTPS, to see readable traffic).

```bash
# Install Wireshark on macOS
brew install wireshark

# Install Wireshark on Ubuntu
sudo apt install wireshark

# Start capturing packets
# Open Wireshark GUI → Interfaces → Select your network adapter → Start capture
```

Visit `http://example.com` and observe what you see in Wireshark:

```
Without VPN, viewing HTTP traffic:
Frame 1: Your IP → example.com server IP
TCP: Your source port (random) → destination port 80
HTTP: GET / HTTP/1.1
       Host: example.com
       User-Agent: Mozilla/5.0...
       (All readable in plaintext)

Frame 2: example.com → Your IP
TCP: Port 80 → Your source port
HTTP: 200 OK
      <html>...</html>
      (Response is readable in plaintext)

Result: Complete visibility into what you're doing
```

### Encrypted (With VPN) - See Only VPN Protocol

Now connect to your VPN and capture the same traffic while visiting HTTPS websites (which use encryption).

```
With VPN, viewing traffic:
Frame 1: Your IP → VPN server IP
[Protocol]: Your source port → destination port (VPN protocol port, e.g., 443)
[Encrypted payload]: (garbled bytes, unreadable)

Frame 2: VPN server IP → Your IP
[Protocol]: VPN protocol port → Your source port
[Encrypted payload]: (garbled bytes, unreadable)

Result: Cannot see destination, content, or what you're doing
```

The key observation: your real IP is gone, replaced by VPN server IP. The destination website is unknown. The payload is gibberish.

### Step 2: Test 1: DNS Leak Detection (Most Common Failure)

DNS is the protocol that translates domain names into IP addresses. Many VPNs encrypt your traffic but leak DNS requests—your ISP can see what websites you visit by watching DNS lookups, even though they cannot see the content.

### Why DNS Leaks Happen

Your operating system has a default DNS resolver (usually your ISP's). Many VPN clients forget to reconfigure it. Result:

```
Traffic encryption: Your IP → VPN IP
DNS resolution: Your IP → ISP's DNS server (unencrypted)

The ISP sees DNS requests like:
client.example.com → 1.2.3.4 (your real IP)
pornhub.com → 1.2.3.4
leaked-secrets.com → 1.2.3.4
```

### Detecting DNS Leaks

**Method 1: Online DNS Leak Test (Easiest)**

Visit https://dnsleaktest.com

```
Steps:
1. Do NOT use VPN yet, note your ISP
2. Connect to VPN
3. Go to dnsleaktest.com
4. Click "Standard test"
5. Observe the DNS servers listed

Result should show:
- VPN's DNS servers (not your ISP)
- All servers owned by VPN provider or trusted DNS providers
- No servers belonging to your ISP

Example:
✓ Good: Query #1: 198.101.242.72 (ExpressVPN DNS)
         Query #2: 198.101.243.72 (ExpressVPN DNS)

✗ Bad:  Query #1: 198.101.242.72 (ExpressVPN DNS)
        Query #2: 8.8.8.8 (Google - okay)
        Query #3: 192.168.1.1 (Your router - LEAK!)
```

**Method 2: CLI DNS Resolution Inspection**

Use `dig` or `nslookup` to see which DNS servers respond to your queries:

```bash
# macOS/Linux: See all DNS servers queried
dig @8.8.8.8 example.com

# Connect to VPN first, then:
# If using Linux with full DNS control:
nmcli dev show | grep DNS
# Should show VPN's DNS servers

# If using macOS:
scutil --dns | grep nameserver
# Should show VPN's DNS servers, not ISP's
```

**Method 3: Wireshark DNS Packet Inspection**

For absolute verification, inspect packets with Wireshark:

```bash
# Capture packets
# Filter by DNS in Wireshark: dns

# Unencrypted DNS (bad):
Frame X: Your IP → ISP DNS (8.8.8.8:53) [UNENCRYPTED]
DNS Query: example.com

# Encrypted DNS (good):
Frame X: Your IP → VPN Server IP (OpenVPN port 443)
[Encrypted payload - cannot see DNS query inside]
```

### Common DNS Leak Scenarios

| Scenario | What Happens | Why | Fix |
|----------|-------------|-----|-----|
| VPN doesn't reconfigure system DNS | DNS requests leak to ISP | VPN client didn't update `/etc/resolv.conf` | Manually set DNS to VPN's or 1.1.1.1 |
| IPv6 DNS | DNS over IPv6 leaks | Only DNS-over-IPv4 is encrypted | Disable IPv6 (see IPv6 test below) |
| DNS fallback | Falls back to system DNS | App queries system DNS after VPN one fails | Configure VPN with multiple DNS servers |
| Split tunnel mode | DNS not encrypted | VPN client doesn't encrypt certain traffic | Disable split tunneling |

### Step 3: Test 2: WebRTC Leak Detection (Sneaky but Preventable)

WebRTC is a protocol for real-time communication (video calls, screen sharing). When WebRTC connects, it leaks your real IP address even when a VPN is active. This is a known vulnerability, not a misconfiguration.

### Why WebRTC Leaks

WebRTC needs to discover your local and public IPs to establish connections. It queries STUN (Simple Traversal of User Datagram Protocol through NAT) servers to determine your real IP. This happens through a side channel the VPN cannot block.

```
What happens without proper VPN:
1. You connect to VPN (IP masked)
2. You visit a website with WebRTC (e.g., video conference)
3. The website's JavaScript calls navigator.mediaDevices.enumerateDevices()
4. WebRTC queries STUN servers
5. STUN response reveals your real IP
6. Website JavaScript reads it: console.log(real_IP)
```

### Detecting WebRTC Leaks

**Method 1: Online WebRTC Leak Test (Easiest)**

Visit https://www.perfect-privacy.com/webrtc-leak-test/

```
Steps:
1. Do NOT use VPN, note your real IP
2. Connect to VPN
3. Go to perfect-privacy.com/webrtc-leak-test/
4. Check if your real IP appears

Result should show:
✓ Good: Only VPN IP visible, not your real IP

✗ Bad: Your real IP appears in WebRTC leak test
```

**Method 2: Browser Console JavaScript Test**

Modern browsers allow you to inspect WebRTC IPs directly:

```javascript
// Open browser console (F12 → Console)
// Run this code:

function findIPAddresses() {
  const ips = new Set();

  const pc = new RTCPeerConnection({
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'stun:stun1.l.google.com:19302' }
    ]
  });

  pc.createDataChannel('');
  pc.createOffer().then(offer => pc.setLocalDescription(offer));

  pc.onicecandidate = (ice) => {
    if (!ice || !ice.candidate) return;

    const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3})/;
    const ipAddress = ipRegex.exec(ice.candidate.candidate)[1];

    if (ipAddress) {
      ips.add(ipAddress);
      console.log('Found IP:', ipAddress);
    }
  };
}

findIPAddresses();
```

Run this, then check if your real IP appears in console. If it does, WebRTC is leaking.

**Method 3: Disable WebRTC in Browsers**

Firefox allows disabling WebRTC:

```
Firefox:
1. Type about:config in address bar
2. Search for media.peerconnection.enabled
3. Set to false (double-click to toggle)
4. Now WebRTC cannot leak

Chrome:
1. Install extension: https://chrome.google.com/webstore/detail/webrtc-leak-prevent...
2. This routes WebRTC through VPN tunnel
```

### Fixing WebRTC Leaks

| Method | What It Does | Effectiveness |
|--------|-------------|----------------|
| Disable WebRTC in browser | Prevents JavaScript from accessing WebRTC | 100% (but breaks video calls) |
| Use VPN with WebRTC leak fix | VPN routes WebRTC through tunnel | 95% (depends on VPN) |
| Disable all plugins | Prevents Flash/Java from exposing IPs | 100% (but breaks some sites) |
| Use browser isolation | Run browser in isolated container | 99% |

Most modern VPNs now prevent WebRTC leaks if configured correctly.

### Step 4: Test 3: Kill Switch Verification (Critical for Security)

A kill switch is a VPN feature that blocks all internet traffic if the VPN connection drops. Without a kill switch, traffic reverts to your real IP unencrypted.

### Why Kill Switches Matter

```
Scenario: VPN connection drops without kill switch

1. You're using VPN, IP masked
2. VPN connection drops (network change, server restart)
3. Your operating system doesn't know about this
4. Traffic routes to default gateway (your ISP)
5. Suddenly your real IP is exposed
6. Websites, ISP, network monitor see your traffic in plaintext
7. VPN reconnects (after 1-10 seconds)
8. But damage is done - your identity was exposed

With kill switch:
1. VPN connection drops
2. Kill switch blocks all traffic immediately
3. No traffic can escape unencrypted
4. VPN reconnects
5. Traffic resumes through VPN
```

### Testing Kill Switch

**Method 1: Simulate VPN Disconnect**

This test shows what happens when VPN fails:

```bash
# macOS: Block VPN IP to simulate disconnect
# Find VPN gateway IP:
netstat -rn | grep tun

# Block traffic to VPN (simulates disconnect):
sudo route add -net [VPN_IP]/32 -blackhole

# Try to access internet:
curl http://example.com

# Expected result with kill switch:
# curl: (7) Failed to connect to example.com port 80: Operation timed out
# (Cannot reach internet without VPN)

# Expected result without kill switch:
# Successfully returns content (using real ISP, leaked!)

# Remove the block:
sudo route delete [VPN_IP]/32
```

**Method 2: Check Kill Switch Setting**

Most VPN clients have kill switch settings. Verify they're enabled:

```
Common VPN apps kill switch location:

ExpressVPN (macOS):
  Menu → Preferences → Network → Network Lock (toggle ON)

ProtonVPN (Linux):
  $ sudo protonvpn kill-switch on

OpenVPN:
  Edit config file, add:
  [block-outside-dns]
  status tcp
  down
```

**Method 3: Actual Connection Drop Test (Advanced)**

This test requires two network devices:

```
Setup:
- Device A: Running VPN, connected to Network B (Wi-Fi)
- Device B: Connected to same Network

Test:
1. On Device A, note your IP with VPN: curl https://whatismyipaddress.com
2. Start monitoring traffic from Device B (tcpdump or Wireshark)
3. Power off Network B's router
4. Try to access internet on Device A for 5 seconds
5. Power router back on
6. Check Device B's captured traffic

Expected with kill switch:
- No unencrypted traffic from Device A during the 5 seconds
- No real IP address visible

Expected without kill switch:
- Device B sees Device A's real IP
- Device B sees unencrypted DNS queries or HTTP requests
```

### Step 5: Test 4: IPv6 Leak Detection

IPv6 is the new internet protocol. Many VPNs only encrypt IPv4 traffic, leaving IPv6 traffic unencrypted. A website using IPv6 can see your real IPv6 address even when you think the VPN is protecting you.

### Understanding IPv6 Leaks

```
Scenario: Partial IPv6 support in VPN

Device IPv4: 1.2.3.4 (masked by VPN)
Device IPv6: 2001:db8::1 (unmasked, real)

Website requests IPv6 address, gets your real IPv6, learns identity
Website also makes IPv4 request, gets VPN's masked IP
Website now knows: masked IPv4 = real IPv6, deanonymizes you
```

### Testing for IPv6 Leaks

**Method 1: Online IPv6 Leak Test**

Visit https://ipv6leak.com

```
Steps:
1. Do NOT use VPN, note your IPv6
2. Connect to VPN
3. Go to ipv6leak.com
4. Check if your real IPv6 is visible

Result should show:
✓ Good: IPv6 address is from VPN provider (or IPv6 is disabled)
✗ Bad: Your real IPv6 address is visible
```

**Method 2: CLI IPv6 Check**

```bash
# See your current IPv6 address:
curl https://ipv6.icanhazip.com

# Connect to VPN, run again:
curl https://ipv6.icanhazip.com

# Expected with VPN:
# Same IPv6 (or different if VPN reassigns)
# NOT your original personal IPv6

# Check your personal IPv6 (before VPN):
ifconfig | grep inet6
# Look for your real IPv6, remember it
# After connecting VPN, run again
# Should NOT match
```

**Method 3: Disable IPv6 If VPN Doesn't Support It**

If your VPN doesn't support IPv6, disable it:

```bash
# macOS: Disable IPv6
networksetup -setv6off Wi-Fi

# Linux: Disable IPv6
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6

# Windows: Open Settings → Network → IPv6 → Disable

# Re-enable later:
networksetup -setv6automatic Wi-Fi  # macOS
echo 0 > /proc/sys/net/ipv6/conf/all/disable_ipv6  # Linux
```

### Step 6: Test 5: Packet Inspection with Wireshark (Complete Verification)

For complete verification, inspect the actual packets your VPN sends:

```bash
# Install Wireshark
brew install wireshark  # macOS
sudo apt install wireshark  # Linux

# Start capturing:
# Open Wireshark → Select network interface → Capture

# Without VPN, visit http://example.com:
# Look for plaintext HTTP packets showing:
#   - Your real IP
#   - Destination IP (example.com)
#   - HTTP GET request (readable)
#   - HTTP response body (readable)

# Connect to VPN, visit https://example.com:
# Look for encrypted packets showing:
#   - Your IP → VPN server IP
#   - Encrypted payload (gibberish)
#   - No visible HTTP content
#   - Destination website not visible

# Filter Wireshark by protocol:
# - OpenVPN: openvpn (UDP port 1194)
# - WireGuard: udp.port == 51820
# - IKEv2: isakmp or esp
```

### Interpreting Wireshark Results

Good VPN packet flow:

```
Frame 1: Client IP:random_port → VPN_IP:1194 [OpenVPN encrypted]
         Encrypted payload (cannot read destination or content)

Frame 2: VPN_IP:1194 → Client IP:random_port [OpenVPN encrypted]
         Encrypted payload

All traffic flows through VPN IP, nothing reveals your real IP or destination
```

Bad VPN packet flow:

```
Frame 1: Client IP:random_port → VPN_IP:1194 [OpenVPN encrypted]
         (Good - traffic encrypted)

Frame 2: Client IP:53 → ISP_DNS:53 [DNS unencrypted]
         example.com A?
         (Bad - DNS leak! Your ISP can see what you're accessing)

Frame 3: Client IP:random_port → example.com:443 [TLS encrypted]
         (Bad - leaking real IP as source)
```

### Step 7: VPN Verification Checklist

Run these tests to completely verify your VPN:

```
Before VPN:
[ ] Test 1: Note your real IP address
    $ curl https://whatismyipaddress.com
[ ] Test 2: Note your real IPv6 address
    $ curl https://ipv6.icanhazip.com
[ ] Test 3: Note your ISP's DNS servers
    Visit dnsleaktest.com, note servers
[ ] Test 4: Check WebRTC IP
    Visit webrtc-leak-test, note IP

Connect to VPN:

[ ] Test 5: Verify IP changed
    $ curl https://whatismyipaddress.com
    (Should NOT be your original IP)

[ ] Test 6: DNS leak test
    Visit dnsleaktest.com
    (Should show VPN DNS, not ISP DNS)

[ ] Test 7: WebRTC leak test
    Visit webrtc-leak-test
    (Should NOT show your real IP)

[ ] Test 8: IPv6 leak test
    $ curl https://ipv6.icanhazip.com
    (Should be different from original IPv6)

[ ] Test 9: Kill switch test
    Kill VPN connection, try to access internet
    (Should fail or show VPN IP, not real IP)

[ ] Test 10: Wireshark inspection
    Capture packets, verify encryption
    (Should see only VPN IP outgoing, all encrypted)

If all tests pass: VPN is properly encrypting your traffic
If any tests fail: VPN has a leak, contact support or switch providers
```

### Step 8: Common VPN Failures and What They Indicate

| Test That Fails | What It Means | Severity |
|-----------------|----------|----------|
| IP unchanged | VPN not connecting at all | Critical |
| DNS leak | VPN not controlling DNS | High |
| WebRTC leak | Website can see real IP | High |
| IPv6 leak | VPN only handles IPv4 | High |
| Kill switch fails | Vulnerable if VPN drops | High |
| Traffic not encrypted (Wireshark) | No actual encryption | Critical |

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Articles

- [How to Verify VPN Is Working Correctly 2026](/privacy-tools-guide/how-to-verify-vpn-is-working-correctly-2026/)
- [How To Use Tcpdump To Verify Vpn Traffic Is Encrypted](/privacy-tools-guide/how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [Verify That Your VPN Is Actually Working and Not Leaking](/privacy-tools-guide/how-to-verify-that-your-vpn-is-actually-working-and-not-leaking/)
- [Verify VPN is Actually Working: DNS, WebRTC, IPv6 Leak Test](/privacy-tools-guide/how-to-verify-vpn-is-actually-working-dns-webrtc-ipv6-leak-test-guide/)
- [Use Tcpdump to Verify VPN Traffic Is Encrypted](/privacy-tools-guide/a140-how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [AI Coding Assistant for Network Traffic Analysis: What](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-network-traffic-analysis-what-connection/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to verify a vpn is actually encrypting your traffic?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
