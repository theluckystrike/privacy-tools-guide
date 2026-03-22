---
title: "How to Configure DNS Over HTTPS (DoH) for Privacy in 2026"
description: "Set up DNS over HTTPS on Windows, macOS, iOS, and Android. Cloudflare, NextDNS, Quad9, and Mullvad compared with configuration examples."
author: Privacy Tools Guide
date: 2026-03-22
reviewed: true
score: 8
voice-checked: true
intent-checked: true
permalink: /how-to-configure-dns-over-https-for-privacy-2026/---
---
title: "How to Configure DNS Over HTTPS (DoH) for Privacy in 2026"
description: "Complete setup guide for DNS over HTTPS on all platforms. Compare Cloudflare, NextDNS, Quad9, and Mullvad. Configuration examples for Windows, macOS, iOS,"
author: Privacy Tools Guide
date: 2026-03-22
reviewed: true
score: 8
voice-checked: true
intent-checked: true
permalink: /how-to-configure-dns-over-https-for-privacy-2026/---

{% raw %}

# How to Configure DNS Over HTTPS (DoH) for Privacy in 2026

Standard DNS queries are unencrypted. Your ISP, network operator, and DNS resolver see every website you visit. DNS over HTTPS (DoH) encrypts DNS lookups, hiding your browsing activity from surveillance. This guide covers DoH setup across all platforms with real configuration steps.

## Key Takeaways

- **Add preferred DNS**: - Preferred: `1.1.1.1`
 - Alternate: `1.0.0.1`
10.
- **Add preferred DNS**: - Preferred: `2606:4700:4700::1111`
 - Alternate: `2606:4700:4700::1001`
12.
- **Username/Password**: Leave blank
7.
- **Install Mullvad VPN app**: (free) 2.
- Toggle "Use custom DNS"
5.

## Why DoH Matters

When you visit example.com, your device sends an unencrypted DNS query: "What is the IP for example.com?" Every network device between you and the DNS resolver can see this query. DoH wraps DNS queries in HTTPS encryption, hiding your traffic from ISPs, corporate networks, and attackers on public WiFi.

**Without DoH:**
```
Your Device → (VISIBLE) → ISP DNS Server
ISP sees: example.com, news.com, banking.com, dating.com (every site)
```

**With DoH:**
```
Your Device → (ENCRYPTED) → DoH Provider
Only ISP sees encrypted traffic (can't identify sites)
Only DoH provider sees domain queries (under their privacy policy)
```

## Cloudflare (1.1.1.1) — Best for Speed and Simplicity

Cloudflare's 1.1.1.1 is the fastest public DoH resolver with strong privacy commitments.

**DoH Address:** https://cloudflare-dns.com/dns-query
**Pricing:** Free
**Privacy Policy:** No query logging, privacy audit published annually

### Windows 10/11 Setup

**Method 1: Settings GUI (Easiest)**

1. Open Settings → Network & internet → Status
2. Scroll down → Advanced network settings
3. Related settings → More network options
4. Click active network → Properties
5. Under "DNS server assignment," click Edit
6. Change from "Automatic" to "Manual"
7. Toggle "IPv4" ON
8. Delete existing DNS servers
9. Add preferred DNS:
 - Preferred: `1.1.1.1`
 - Alternate: `1.0.0.1`
10. Toggle "IPv6" ON
11. Add preferred DNS:
 - Preferred: `2606:4700:4700::1111`
 - Alternate: `2606:4700:4700::1001`
12. Click Save

**Verify:** Open Command Prompt, run:
```
nslookup example.com 1.1.1.1
```

Should resolve with 1.1.1.1 shown as server.

**Method 2: Command Line (Advanced)**

```powershell
# Set IPv4 DNS
Set-DnsClientServerAddress -InterfaceIndex (Get-NetAdapter | Where-Object {$_.Status -eq "Up"}).InterfaceIndex -ServerAddresses ("1.1.1.1", "1.0.0.1")

# Set IPv6 DNS
Set-DnsClientServerAddress -InterfaceIndex (Get-NetAdapter | Where-Object {$_.Status -eq "Up"}).InterfaceIndex -ServerAddresses ("2606:4700:4700::1111", "2606:4700:4700::1001")

# Verify
Get-DnsClientServerAddress
```

### macOS Setup

**System Settings Method:**

1. System Settings → Network → Wi-Fi (or Ethernet)
2. Click Advanced (bottom right)
3. Select DNS tab
4. Click + to add server
5. Enter: `1.1.1.1`
6. Click + again, add: `1.0.0.1`
7. For IPv6 (same DNS tab):
 - Click +
 - Enter: `2606:4700:4700::1111`
 - Click +
 - Enter: `2606:4700:4700::1001`
8. Click OK

**Verify DNS over HTTPS:**

```bash
# Check current DNS
networksetup -getdnsservers Wi-Fi

# Verify DoH works (should resolve instantly)
dig @1.1.1.1 example.com +short
```

### iOS Setup

**Automatic Profile Method (Easiest):**

1. Visit https://1.1.1.1 on iPhone
2. Tap "Install" when prompted
3. Settings → VPN & Device Management
4. Select "1.1.1.1" profile
5. Tap "Install" (top right)
6. Confirm with Face ID
7. Done — DoH active

**Manual Configuration Method:**

1. Settings → General → VPN & Device Management
2. Add configuration manually
3. Protocol: DNS
4. Server: `1.1.1.1`
5. Domain: Leave blank
6. Username/Password: Leave blank
7. Save and enable

### Android Setup

**Method 1: Android 9+ (Built-in DoH)**

1. Settings → Network & internet → Private DNS
2. Select "Private DNS provider hostname"
3. Enter: `cloudflare-dns.com`
4. Save

**Method 2: All Android Versions (App)**

1. Download "1.1.1.1: Faster Internet" from Play Store
2. Open app, tap "Get Started"
3. Tap "Connect" to activate
4. Toggle "DoH" ON
5. Keep app running in background

**Verify:** Open app → Status should show "Connected" with Green checkmark.

## NextDNS — Best for Customization and Control

NextDNS offers granular filtering: block ads, malware, adult content, specific domains, or entire categories.

**DoH Address:** https://dns.nextdns.io/[YOUR_CONFIG_ID]
**Pricing:** Free tier (300k queries/month); Pro at $2/month (unlimited queries, advanced features)
**Privacy:** No persistent logging, configuration stored encrypted

### Setup Steps

**1. Create Account:**
- Visit https://nextdns.io
- Sign up with email
- Confirm email
- Create configuration (auto-assigned ID: e.g., "a1b2c3")

**2. Windows Configuration:**

```
Preferred IPv4 DNS: 45.90.28.0
Alternate IPv4 DNS: 45.90.30.0

Preferred IPv6 DNS: 2a07:a8c0::0
Alternate IPv6 DNS: 2a07:a8c1::0
```

(Replace IPv4 addresses with your NextDNS config ID — check NextDNS dashboard for exact IPs)

**3. iOS/macOS Configuration:**

1. Visit https://nextdns.io from your device
2. Click "Set up NextDNS"
3. Select device type (iPhone, Mac, etc.)
4. Tap "Install Configuration"
5. Confirm with Face ID/password
6. Select which features to enable:
 - Block ads: Yes
 - Block malware: Yes
 - Block adult content: Yes
 - Advanced threat protection: Optional (paid)

**4. Real Example: Create Custom Blocklist**

In NextDNS dashboard:

```
Categories to block:
□ Ads & Trackers
□ Social Media
□ Streaming Services
□ News
□ Adult Content
□ Malware
□ Phishing

Custom domains to always block:
- facebook.com
- tiktok.com
- instagram.com

Custom domains to always allow:
- linkedin.com (whitelist for work)
- youtube.com (whitelist despite streaming category)
```

Result: DNS queries for blocked domains return NXDOMAIN (site unreachable), effectively blocking them network-wide.

**Cost Analysis:**
- Free tier: $0 (fine for personal use, 300k queries/month = ~10k/day)
- Pro: $2/month (unlimited, advanced features, device management)
- Typical home user: 100k-200k queries/month (1 free tier sufficient)

## Quad9 — Best for Security-Focused Users

Quad9 emphasizes malware and phishing protection. Specifically designed for security rather than just privacy.

**DoH Address:** https://dns.quad9.net/dns-query
**Pricing:** Free
**Privacy Policy:** No logging, independent security audits published

### Setup

**Windows:**
```
Preferred: 9.9.9.9
Alternate: 149.112.112.112

IPv6:
Preferred: 2620:fe::fe
Alternate: 2620:fe::9
```

**macOS/iOS/Android:**
Visit https://www.quad9.net/setup and follow device-specific instructions.

**Key Features:**
- Blocks malicious domains in real-time
- Threat intelligence from multiple sources
- Optional DNSSEC validation (strict mode)
- Works worldwide with 200+ POPs

## Mullvad — Best for Extreme Privacy

Mullvad runs the DoH resolver as part of their privacy-first operation. Maximum privacy.

**DoH Address:** https://dns.mullvad.net/dns-query
**Pricing:** Free (part of Mullvad VPN infrastructure)
**Privacy:** Zero logging, no activity history retained

### Setup

**Simple Method (Recommended):**
1. Install Mullvad VPN app (free)
2. Open Mullvad
3. Settings → DNS
4. Toggle "Use custom DNS"
5. Add: `dns.mullvad.net/dns-query`
6. Done — now using DoH through Mullvad

**Advanced Method (Without VPN):**
```
Preferred: 194.242.2.2
Alternate: 194.242.2.3

IPv6:
Preferred: 2a07:e340::1
Alternate: 2a07:e340::2
```

## Provider Comparison Table

| Provider | Free | Speed | Logging | Filtering | Security | Customization |
|----------|------|-------|---------|-----------|----------|---------------|
| Cloudflare | Yes | Fastest | None | Limited | Good | Low |
| NextDNS | Yes | Fast | None | Excellent | Excellent | High |
| Quad9 | Yes | Fast | None | Excellent (malware) | Excellent | Low |
| Mullvad | Yes | Good | None | Limited | Excellent | Medium |

## Verify DoH is Working

**Method 1: DNS Query Tool**

```bash
# Test with dig (Linux/macOS)
dig @1.1.1.1 example.com

# Test with nslookup (Windows)
nslookup example.com 1.1.1.1

# Should return IP addresses, confirming DNS works
```

**Method 2: Online Test**

Visit https://1.1.1.1/help and check:
- DNS Provider: Should show "Cloudflare" or your chosen provider
- Using HTTPS: Should show "Yes"
- DNSSEC: Should show "Yes" (validated)

**Method 3: Check Encryption**

```bash
# macOS/Linux — capture DNS traffic (should be encrypted)
tcpdump -i en0 'port 53'
# Should NOT show unencrypted DNS queries if DoH is active
```

## Performance Considerations

DoH adds minimal overhead (~1-5ms per query) but:
- Faster ISP DNS might be ~0ms (local)
- Cloudflare 1.1.1.1: Globally optimized, typically <10ms
- NextDNS: Varies by location, usually 10-20ms
- Mullvad: ~20-30ms (conservative approach)

**Real-world impact:** Unnoticeable for daily browsing. Page loads nearly identical.

## Privacy Risks Still Remaining

DoH solves ISP tracking, but consider:

**What DoH doesn't hide:**
- Your IP address (visible to website servers)
- HTTPS connections still show destination IP (if not SNI-encrypted)
- DoH provider itself sees domains queried (choose provider carefully)

**Complete privacy requires:**
1. DoH (hides ISP tracking) ✓
2. VPN (hides IP address, location)
3. HTTPS (encrypts website content)
4. DNSSec validation (prevents DNS hijacking)

## Troubleshooting DoH Issues

**Problem: Some websites won't load**
- Solution: Add exception to blocklist (NextDNS) or switch to Cloudflare (minimal blocking)

**Problem: Slow performance**
- Solution: Switch to Cloudflare 1.1.1.1 (fastest) or reduce filtering in NextDNS

**Problem: Apps stop working after enabling DoH**
- Solution: Create whitelist exceptions or temporarily disable for specific apps

**Problem: DoH conflicts with corporate network**
- Solution: Use Cloudflare (corporate-friendly) or ask IT for exceptions

## Recommended Configuration for Most Users

```
Primary DNS: Cloudflare 1.1.1.1 (speed and simplicity)
Backup DNS: Quad9 (security, malware blocking)
Extra layer: NextDNS Pro if concerned about tracking ($2/month)
```

Cost: Free to $2/month
Setup time: 5-15 minutes across all devices
Privacy improvement: Eliminates ISP DNS logging (significant)

## Related Articles

- [VPN vs DNS Privacy: Which Do You Really Need](/articles/vpn-vs-dns-privacy-2026.md)
- [DNSSEC Validation and Your Privacy](/articles/dnssec-validation-privacy-2026.md)
- [How ISPs Track Your Browsing Without VPN](/articles/isp-tracking-without-vpn-2026.md)
- [Best Privacy-Focused VPN Providers 2026](/articles/best-privacy-vpn-2026.md)
- [Network-Wide Ad Blocking with Pi-hole and DoH](/articles/pihole-doh-blocking-2026.md)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
