---
layout: default
title: "How to Verify VPN Is Working Correctly 2026"
description: "Step-by-step tests for DNS leaks, WebRTC leaks, IPv6 leaks, and kill switch verification to confirm your VPN is actually protecting you"
date: 2026-03-21
last_modified_at: 2026-03-21
author: "Privacy Tools Guide"
permalink: /how-to-verify-vpn-is-working-correctly-2026/
categories: [guides]
tags: [privacy-tools-guide, vpn, verification, networking]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

You pay for a VPN monthly and assume it's protecting your traffic. But VPN companies are not immune to configuration errors, and many VPNs have critical flaws: DNS leaks revealing browsing history, WebRTC leaks exposing real IP addresses, IPv6 leaks bypassing encryption entirely, and kill switches that don't trigger.

This guide walks through eight verification tests you can run right now to confirm your VPN is actually working. No technical background required. All tools are free.

## VPN Security Model: What You're Testing

A VPN tunnels all traffic through an encrypted pipe to a VPN server, which forwards requests to the internet. Your ISP sees encrypted data; your ISP cannot see what sites you visit.

However, misconfigurations break this model:

- **DNS leak:** VPN sends traffic through encrypted tunnel, but DNS queries go to ISP's DNS server. ISP sees every site you visit.
- **WebRTC leak:** Browser's WebRTC protocol bypasses VPN tunnel, exposing real IP address directly.
- **IPv6 leak:** You have an IPv6 address outside the VPN tunnel while using IPv4 inside it. Attackers can identify you via IPv6.
- **Kill switch failure:** Internet connection remains active if VPN disconnects. Real IP address becomes visible.

These are not theoretical; they're common. A 2016 study found 38% of VPNs leaked DNS. Grab a free testing tool and verify your VPN today.

---

## Test 1: DNS Leak Detection (Most Common)

**What it tests:** Whether DNS queries go through the VPN or leak to ISP.

**How to test:**

1. **Before VPN:** Visit ipleak.net in browser (VPN disabled).
   - Note the "Your DNS servers" entries (typically ISP addresses like 1.1.1.1, 8.8.8.8).

2. **Connect VPN:** Turn on your VPN, choose a server location.

3. **After VPN:** Revisit ipleak.net.
   - DNS servers should change to VPN provider's servers.
   - If DNS servers are still ISP addresses, **you have a DNS leak**.

**Example:**
- Before VPN: DNS = 71.252.0.0 (Comcast ISP)
- After VPN: DNS = 10.8.0.1 (VPN provider)
- Result: **PASS**

**If you see a leak:**
- On Windows: Settings > Network > VPN > Advanced > DNS settings. Disable NRPT (Name Resolution Policy Table) or use manual DNS.
- On Mac: System Preferences > Network > VPN > Advanced > DNS. Ensure VPN DNS is listed first.
- On Linux: Edit `/etc/resolv.conf` to use VPN DNS, or use `resolvectl` to override system DNS.

**ipleak.net verification:** Free, no signup. Takes 30 seconds.

---

## Test 2: WebRTC Leak Detection (Browser Vulnerability)

**What it tests:** Whether browser's WebRTC protocol leaks your real IP address even with VPN connected.

**How to test:**

1. **Before VPN:** Visit ipleak.net/webrtc (VPN disabled).
   - Note the "Your IP addresses" listed (your real IP).

2. **Connect VPN.**

3. **After VPN:** Revisit ipleak.net/webrtc.
   - "Your IP addresses" should only show VPN server IP.
   - If your real IP appears, **you have a WebRTC leak**.

**Example:**
- Before VPN: IP = 203.0.113.45 (your real address)
- After VPN: IP = 185.199.108.200 (VPN server)
- Result: **PASS**

**If you see a leak:**
- **Chrome/Brave:** Disable WebRTC in chrome://flags (search "webrtc", toggle to disabled). May break video conferencing.
- **Firefox:** Enter about:config, search "media.peerconnection.enabled", set to false.
- **Safari:** WebRTC leaks are common in Safari; disable if privacy is critical, use Firefox instead.
- **Electron apps (Discord, Slack):** No direct WebRTC control; these apps are at risk. Consider using web versions instead.

** test:** ipleak.net has a more thorough WebRTC test. Click "Show my IPv4 + IPv6 + DNS + WebRTC leak status". but slower.

---

## Test 3: IPv6 Leak Detection

**What it tests:** Whether you're leaking an IPv6 address outside the VPN tunnel.

**How to test:**

1. **Disable VPN.** Visit ipleak.net/ipv6.
   - Note if an IPv6 address is displayed.
   - If no IPv6 is shown, your network doesn't support IPv6; skip this test.

2. **Enable VPN.**

3. **Revisit ipleak.net/ipv6.**
   - If an IPv6 address is shown, **you have an IPv6 leak**.
   - IPv6 should be blocked or routed through VPN.

**Example:**
- Before VPN: IPv6 = 2001:db8::1 (your real IPv6)
- After VPN: IPv6 = (no address) OR IPv6 = 2001:db8::vvv (VPN's IPv6)
- Result: **PASS**

**If you see a leak:**
- Most VPNs don't properly route IPv6. Solutions:
  - **Disable IPv6 on your system** (simplest). On Windows: Device Manager > Network adapters > IPv6 > disable. On Mac: System Preferences > Network > IPv6 > Off. On Linux: `echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6`.
  - **Use VPN kill switch:** If VPN doesn't fully support IPv6, kill switch should block all IPv6 traffic.

---

## Test 4: Real IP Address Leak (IP Geolocation Check)

**What it tests:** Whether your real IP address and location are visible.

**How to test:**

1. **Disable VPN.** Visit ipify.org or whatismyipaddress.com.
   - Note your IP address and city/country.

2. **Connect VPN to a server in a different country** (e.g., if you're in US, choose Japan server).

3. **Revisit the site.**
   - IP should change to VPN server IP.
   - City/country should match VPN server location.
   - If your real IP or location is visible, **you have a leak**.

**Example:**
- Before VPN: IP = 203.0.113.45 (Los Angeles, USA)
- After VPN to Japan server: IP = 203.0.114.100 (Tokyo, Japan)
- Result: **PASS**

**If you see your real IP:**
- Reconnect VPN.
- Clear browser cookies (Ctrl+Shift+Delete).
- Try a different VPN server location.
- If leak persists, contact VPN provider support.

---

## Test 5: Kill Switch Verification (Critical Test)

**What it tests:** Whether your kill switch actually blocks all traffic if VPN disconnects.

**Warning:** This test temporarily breaks your internet connection. Save your work first.

**How to test:**

1. **Start a download** that will take 30+ seconds (e.g., torrent, large file).
2. **Connect VPN.**
3. **Let download start.** Confirm VPN is connected.
4. **Disconnect VPN manually** or unplug network cable.
   - If kill switch works: Download stops immediately; internet goes dark.
   - If kill switch fails: Download continues on real IP; **major security breach**.

**Better alternative (no download):**

1. **Connect VPN.**
2. **Start streaming video** on ipleak.net or run continuous ping in terminal.
3. **Unplug VPN network interface** (if on separate device/VM).
4. **Observe:**
   - Kill switch ON: Video stops, ping fails. Good.
   - Kill switch OFF: Video continues or ping resumes on real IP. Bad.

**macOS Command-Line Test:**
```bash
# While VPN is connected:
ping 8.8.8.8

# Stop VPN suddenly and observe ping output
# Kill switch ON: Ping fails immediately
# Kill switch OFF: Ping continues with high latency (real IP)
```

**If kill switch fails:**
- Most VPN apps have kill switch in Settings > Security > "Kill Switch" toggle.
- Enable it.
- Restart VPN.
- Re-test.

---

## Test 6: VPN Disconnection Detection (Browser Leak Test)

**What it tests:** Whether your browser is aware of VPN disconnection.

**How to test:**

1. **Connect VPN.**
2. **Open ipleak.net in browser.**
3. **Force disconnect VPN** (unplug network, turn off WiFi, or kill VPN app).
4. **Refresh ipleak.net page.**
   - If kill switch works, page won't load (internet is blocked).
   - If kill switch fails, page loads and shows your real IP. **Leak.**

**Alternative:** Use BrowserLeaks.com for leak testing. It tests:
- IP address
- WebRTC
- DNS
- Browser fingerprinting

All in one test. Very reliable.

---

## Test 7: DNS Leak Over Time (Passive Test)

**What it tests:** Whether DNS leaks occur intermittently.

**How to test:**

1. **Connect VPN.**
2. **Open Command Prompt / Terminal.**
3. **Run DNS query test repeatedly:**

**Windows:**
```cmd
nslookup google.com
nslookup google.com
nslookup google.com
```

**macOS/Linux:**
```bash
dig @8.8.8.8 google.com
dig @8.8.8.8 google.com
dig @8.8.8.8 google.com
```

4. **Observe response times.**
   - Consistent fast response: Likely going through VPN DNS cache.
   - Intermittent slow response: Possible fallback to ISP DNS.

**Advanced:** Use Wireshark (packet sniffer) to monitor DNS traffic. If you see DNS queries to non-VPN addresses, you have a leak. Requires technical knowledge; ipleak.net is simpler.

---

## Test 8: Kill Switch Test with Multiple VPN Servers

**What it tests:** Whether kill switch works across all VPN servers.

**How to test:**

1. Connect to VPN Server A (e.g., US).
2. Run kill switch test (disconnect VPN, check if internet blocks).
3. Reconnect to Server B (e.g., Japan).
4. Repeat kill switch test.
5. Try a third server.

**Why:** Some VPNs have server-specific issues. Kill switch might work on US servers but fail on Japanese servers.

**If you see variance:**
- Contact VPN support with specific server locations.
- Use only servers where kill switch verified.

---

## VPN Brands: Known Leak Issues (2026)

**Generally safe (rare leaks):**
- Mullvad VPN (independent audit, IPv6 support)
- IVPN (transparent, based in Gibraltar)
- ProtonVPN (owned by Swiss company Proton, audited)

**Known issues:**
- NordVPN: WebRTC leaks reported 2020–2022 (likely fixed, still verify)
- ExpressVPN: DNS leaks in some regions
- CyberGhost: IPv6 leaks reported
- Windscribe: Kill switch occasionally fails (known issue)

**Always verify your specific VPN.** Don't trust brand reputation alone.

---

## Quick Test: 5-Minute Verification

If you only have 5 minutes:

1. Open ipleak.net in browser (VPN disconnected). Note ISP DNS and IP.
2. Connect VPN. Revisit ipleak.net.
3. Confirm DNS changed to VPN provider's DNS.
4. Confirm IP changed to VPN server IP.
5. Disable VPN suddenly (unplug network). Try to refresh page.
6. If internet blocks, kill switch works. **PASS.**

That's 90% of VPN verification in 5 minutes.

---

## Advanced: Monitoring VPN Leaks (Automated)

If you want continuous monitoring:

**Option 1: CyberGhost's leak test (desktop app):**
- Built-in leak test in CyberGhost VPN app.
- Runs automatically on connect.
- Shows results in app dashboard.

**Option 2: VPN leak monitoring service:**
- VPN Leak Test API (vpnleaktest.com/api)
- Automated checks via cron job.
- Alerts if leak detected.

**Option 3: Custom monitoring (Linux):**
```bash
#!/bin/bash
# Monitor for DNS leaks hourly

while true; do
  echo "Testing for DNS leaks..."
  curl -s https://ipleak.net/json | jq '.dns_leak'
  sleep 3600  # Check every hour
done
```

---

## Verdict: Trust But Verify

VPN providers are not adversarial, but they're also not immune to mistakes. A DNS leak, WebRTC leak, or kill switch failure is a single bug away.

Run these tests monthly. More often if you change VPN providers or update your OS.

A VPN is only as good as its actual implementation. Test it.

{% endraw %}
