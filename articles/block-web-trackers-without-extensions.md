---
layout: default
title: "How to Block Web Trackers Without Extensions"
description: "Use DNS-level blocking, browser settings, and system-wide hosts files to stop web trackers without installing browser extensions or ad blockers"
date: 2026-03-22
author: theluckystrike
permalink: /block-web-trackers-without-extensions/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Block Web Trackers Without Extensions

Browser extensions are the standard advice for blocking trackers, but they come with trade-offs: they can see everything you browse, they add attack surface, and some have had vulnerabilities. DNS-level blocking, system hosts files, and browser native settings catch most trackers without adding extensions to your browser.

This guide covers five layers of tracker blocking — from DNS resolvers and local Pi-hole deployments, to browser-native privacy configuration, advanced Firefox preferences, and verification tools to confirm that blocking is actually working.
---

## Table of Contents

- [Layer 1: DNS-Level Blocking](#layer-1-dns-level-blocking)
- [Layer 2: System-Wide Hosts File](#layer-2-system-wide-hosts-file)
- [Layer 3: Browser Native Privacy Settings](#layer-3-browser-native-privacy-settings)
- [Layer 4: Firefox Advanced Config](#layer-4-firefox-advanced-config)
- [Layer 5: Verify Tracker Blocking](#layer-5-verify-tracker-blocking)
- [HTTPS Inspection for Full Visibility](#https-inspection-for-full-visibility)
- [What This Approach Misses](#what-this-approach-misses)
- [Putting It Together: Coverage by Layer](#putting-it-together-coverage-by-layer)
- [Layer 6: Advanced DNS Techniques](#layer-6-advanced-dns-techniques)
- [Layer 7: Content Security Policy Inspection](#layer-7-content-security-policy-inspection)
- [Layer 8: Monitoring Your Success](#layer-8-monitoring-your-success)
- [ISP-Level Visibility You Cannot Block](#isp-level-visibility-you-cannot-block)
- [Performance Impact of DNS Blocking](#performance-impact-of-dns-blocking)
- [Bypassing Advanced Tracker Evasion](#bypassing-advanced-tracker-evasion)
- [Related Reading](#related-reading)

## Layer 1: DNS-Level Blocking

DNS blocking works by refusing to resolve tracker domains. When a page tries to load `googletagmanager.com`, your DNS resolver returns NXDOMAIN (no such domain) instead of the real IP. The request never leaves your network.

### Option A: Use a Privacy-Respecting DNS Resolver

| Resolver | Address | Blocks trackers? |
|----------|---------|-----------------|
| NextDNS | Configurable | Yes (custom lists) |
| Quad9 | 9.9.9.9 | Malware only |
| AdGuard DNS | 94.140.14.14 | Trackers + ads |
| Mullvad DNS | 194.242.2.3 | Ads + trackers |

```bash
# Set system DNS on Linux (systemd-resolved)
sudo resolvectl dns eth0 94.140.14.14 94.140.15.15
sudo resolvectl dnssec eth0 yes

# macOS
networksetup -setdnsservers Wi-Fi 94.140.14.14 94.140.15.15

# Verify
resolvectl status eth0
scutil --dns | grep nameserver
```

AdGuard DNS (94.140.14.14) blocks roughly 90,000 tracker and ad domains by default. Mullvad DNS (194.242.2.3) is operated by the VPN provider and logs no queries. NextDNS is different in that it is configurable per-account — you get a unique resolver address, can add blocklists, and get query analytics. All three support DNS-over-HTTPS and DNS-over-TLS for encrypted queries.

To enable DNS-over-HTTPS in systemd-resolved (systemd 247+):

```bash
# /etc/systemd/resolved.conf
[Resolve]
DNS=94.140.14.14
DNSOverTLS=yes
DNSSEC=yes
```

```bash
sudo systemctl restart systemd-resolved
resolvectl status | grep "DNS over TLS"
```

### Option B: Run Pi-hole Locally

```bash
# Install Pi-hole
curl -sSL https://install.pi-hole.net | bash

# Add additional blocklists in Pi-hole admin → Adlists:
# https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
# https://blocklistproject.github.io/Lists/tracking.txt
# https://v.firebog.net/hosts/Easyprivacy.txt

# Update gravity (download blocklists)
pihole -g

# Check statistics
pihole -c
```

Pi-hole blocks ~20-25% of all DNS queries on average home networks. The combination of Steven Black's consolidated hosts list (~130,000 entries), the Firebog Easyprivacy list (~100,000 tracking domains), and the BlocklistProject tracking list covers the overwhelming majority of known third-party trackers.

Pi-hole runs on any Linux machine including a Raspberry Pi Zero W and acts as the DNS server for your entire network when you set it as your router's upstream DNS. Every device on the network — phones, smart TVs, game consoles — benefits without any per-device configuration.

---

## Layer 2: System-Wide Hosts File

```bash
# Download Steven Black's hosts file
sudo curl -o /etc/hosts https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

# Verify
cat /etc/hosts | grep "googletagmanager.com"
# Should show: 0.0.0.0 googletagmanager.com

# Keep it updated (add to crontab)
echo "0 3 * * 0 root curl -s https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts > /etc/hosts" \
  | sudo tee /etc/cron.d/update-hosts
```

The hosts file approach works independently of your DNS resolver — it takes effect even if you are using an ISP DNS that resolves everything. Entries in `/etc/hosts` are checked first by the OS resolver before any DNS query goes out.

The main limitation is lookup performance at scale: a hosts file with 130,000 entries creates a linear search on most systems. Modern Linux with nscd or systemd-resolved caches results, so the practical performance impact is minimal after the first lookup. On older systems or embedded devices, stick with DNS-level blocking instead.

---

## Layer 3: Browser Native Privacy Settings

### Firefox

```
about:config → search "privacy.trackingprotection"

privacy.trackingprotection.enabled = true
privacy.trackingprotection.socialtracking.enabled = true
privacy.trackingprotection.fingerprinting.enabled = true
privacy.trackingprotection.cryptomining.enabled = true
network.cookie.cookieBehavior = 5   (total cookie protection)
privacy.firstparty.isolate = true
```

**Firefox Enhanced Tracking Protection (UI):**
```
about:preferences#privacy → Enhanced Tracking Protection → Strict
```

Firefox's built-in tracker blocking uses the Disconnect.me list and Mozilla's own analytics, and covers social trackers from Facebook, Twitter, LinkedIn, and Pinterest. `network.cookie.cookieBehavior = 5` enables Total Cookie Protection, which partitions cookies per site — a cookie set by `facebook.com` while you visit a site with a Like button is stored separately for each top-level site you visit and cannot be used to track you across sites.

### Chrome/Chromium

```
Settings → Privacy and security → Third-party cookies → Block third-party cookies
Settings → Privacy and security → Privacy Sandbox → turn off all
```

### Safari

```
Settings → Privacy → Prevent cross-site tracking: ON
Settings → Privacy → Hide IP address from trackers: ON
```

Safari's Intelligent Tracking Prevention (ITP) uses on-device machine learning to classify domains as trackers and partitions or purges their storage. "Hide IP address from trackers" routes requests to known tracker domains through Apple's Private Relay infrastructure, masking your real IP.

---

## Layer 4: Firefox Advanced Config

```
Firefox → about:config:
network.http.referer.XOriginPolicy = 2  # Don't send Referer cross-origin
dom.storage.enabled = false             # Breaks some sites but blocks storage tracking
```

Additional preferences worth setting in Firefox for a hardened configuration:

```
privacy.resistFingerprinting = true
# Reports standardized values for: screen resolution, timezone, font list,
# canvas fingerprint, WebGL renderer, hardware concurrency

geo.enabled = false
# Disables the Geolocation API entirely

media.peerconnection.enabled = false
# Disables WebRTC, which can leak local IP addresses even through VPNs

network.http.sendRefererHeader = 1
# Sends Referer only for same-site navigation (1) vs always (2)
```

`privacy.resistFingerprinting` reduces your browser's uniqueness by spoofing system values that sites use for fingerprinting. The trade-off is some sites detect it and present a degraded experience. Test with `coveryourtracks.eff.org` before and after to see how it affects your fingerprint uniqueness score.

---

## Layer 5: Verify Tracker Blocking

```bash
# Test DNS blocking — these should fail to resolve
nslookup googletagmanager.com    # Returns NXDOMAIN if blocked
nslookup doubleclick.net
nslookup connect.facebook.net

# Online tests:
# https://coveryourtracks.eff.org  — EFF's tracker test
# https://browserleaks.com         — leak tests
```

The EFF's Cover Your Tracks test reports three things: whether your browser blocks tracking ads, whether it blocks invisible trackers, and whether your browser fingerprint is unique. Running this before and after each layer of configuration gives a concrete measure of improvement.

For DNS-level blocking specifically, the fastest verification is:

```bash
nslookup googletagmanager.com 127.0.0.1   # Test against local Pi-hole
# Should return: ** server can't find googletagmanager.com: NXDOMAIN
```

---

## HTTPS Inspection for Full Visibility

```bash
# mitmproxy — intercept and filter HTTPS traffic
pip install mitmproxy

# Start in filter mode
mitmproxy --mode regular --ssl-insecure

# Set browser proxy to 127.0.0.1:8080
# Install mitmproxy CA certificate in browser

# View all requests including "hidden" tracker calls
mitmweb --mode regular
```

mitmproxy's web interface (`mitmweb`) shows every request in real time, including HTTPS traffic. Filter with `~d googletagmanager.com` to isolate traffic to specific domains. This technique reveals trackers that your DNS blocking should be catching and any that are slipping through — for example, first-party trackers hosted on the same domain as the content.

---

## What This Approach Misses

DNS and hosts blocking does not cover:

- **First-party tracking**: Trackers hosted on the same domain as the site
- **CNAME cloaking**: Trackers that use the site's own subdomain as a CNAME alias
- **JS fingerprinting**: Runs entirely in the page, no outbound network call to block

For fingerprinting protection, Firefox with `privacy.resistFingerprinting = true` reduces the uniqueness of your browser signature by reporting standardized values for screen size, fonts, and timezone.

CNAME cloaking is worth understanding in detail. A site can configure `tracking.example.com` as a CNAME pointing to `tracker-vendor.com`. Your DNS blocker sees a request for `tracking.example.com` — which is the site's own subdomain and not on any blocklist — and resolves it. Some DNS-over-HTTPS resolvers (NextDNS, AdGuard DNS) follow the CNAME chain and block based on the final destination. Standard resolvers do not.

---

## Putting It Together: Coverage by Layer

The following table shows which tracker types each layer addresses, so you can choose the right combination for your threat model:

| Tracker type | DNS blocking | Hosts file | Browser settings | Firefox advanced |
|---|---|---|---|---|
| Third-party ad trackers | Blocks most | Blocks most | Yes | Yes |
| Social widgets (Like buttons) | Yes | Yes | Yes (strict mode) | Yes |
| Analytics (Google Analytics) | Yes | Yes | Partial | Yes |
| CNAME-cloaked trackers | Partial (NextDNS) | No | No | No |
| First-party analytics | No | No | No | No |
| Canvas/font fingerprinting | No | No | Partial | Yes (RFP) |
| WebRTC IP leak | No | No | No | Yes (disable WebRTC) |

For most users, combining AdGuard DNS (or Pi-hole) with Firefox in Strict mode and `privacy.resistFingerprinting = true` provides strong tracker coverage. Adding the system hosts file gives redundant coverage that survives DNS configuration changes. The mitmproxy inspection step is optional but recommended at least once to understand what is and is not being blocked on the sites you visit regularly.

---

## Layer 6: Advanced DNS Techniques

### DNS-over-HTTPS (DoH) for Enforced Privacy

Standard DNS queries are sent in plaintext—your ISP can see every domain you request. DNS-over-HTTPS encrypts DNS queries end-to-end:

```bash
# Firefox: about:config
network.trr.mode = 2  # Strict DoH mode
network.trr.uri = https://94.140.14.14/dns-query  # AdGuard DoH endpoint

# Linux with systemd-resolved
cat > /etc/systemd/resolved.conf <<EOF
DNS=94.140.14.14
FallbackDNS=94.140.15.15
DNSSEC=yes
DNSSECNegativeTrustAnchors=
EOF
systemctl restart systemd-resolved
```

DoH prevents ISP-level visibility of your DNS queries. However, DoH providers can still see what you're querying. Pair DoH with a privacy-respecting resolver like Mullvad or CENO for stronger guarantees.

### Stubby for Local DNS Encryption

Stubby is a DNS over TLS resolver you run locally:

```bash
# Install Stubby
sudo apt install stubby
# macOS
brew install stubby

# Configure /etc/stubby/stubby.yml
upstream_recursive_servers:
  - address_data: 94.140.14.14
    tls_auth_name: "dns.adguard.com"
    tls_pubkey_pinning: "ABCD1234..."

# Start Stubby daemon
sudo systemctl start stubby
sudo systemctl enable stubby

# Test
dig @127.0.0.1 example.com
```

Stubby encrypts all DNS traffic between your machine and the resolver. This adds a security layer but requires local daemon management.

## Layer 7: Content Security Policy Inspection

Modern sites implement Content Security Policy (CSP) headers that restrict where scripts and resources load from. Examining CSP reveals the tracking infrastructure a site uses:

```bash
# Check CSP headers
curl -I https://example.com | grep -i "content-security"

# Example CSP output:
# Content-Security-Policy: default-src 'self'; script-src 'self' *.example.com *.analytics.com

# Domains after script-src are allowed to inject JavaScript
# Domains referenced in img-src, font-src, etc. can load tracking pixels/fonts
```

Extract and block these domains at the DNS level for that specific site.

## Layer 8: Monitoring Your Success

Verify that your blocking is actually working:

```bash
# Create a simple test page to check what resolves
cat > tracker-test.html <<EOF
<html>
<head>
<script src="https://googletagmanager.com/gtag/js?id=G-XXXXX"></script>
<img src="https://doubleclick.net/test.gif"/>
<script src="https://connect.facebook.net/test.js"></script>
</head>
<body>Open developer console (F12) and check Network tab</body>
</html>
EOF

# Browser should show failed requests for tracker domains
# If requests succeed, your blocking isn't working
```

## ISP-Level Visibility You Cannot Block

Even with all these layers, your ISP still sees:
- **IP addresses you connect to** (via BGP routing)
- **Broad site categories** (you're on netflix.com, regardless of page)
- **Encrypted SNI** (Server Name Indication) if not using ESNI/ECH

For ISP privacy, you need a VPN, Tor, or ESNI-enabled browser:

```bash
# Enable ESNI/ECH in Firefox (2026+)
about:config → search "ech"
network.dns.use_https_rr_for_ech = true
```

ESNI hides the specific hostname from your ISP, showing only the IP address.

## Performance Impact of DNS Blocking

DNS blocking introduces minimal latency (1-5ms per query) but can slow initial page loads if the blocklist is misconfigured:

```bash
# Benchmark Pi-hole DNS performance
for i in {1..10}; do
    time dig example.com @192.168.1.1
done | grep real

# Compare against unblocked DNS
for i in {1..10}; do
    time dig example.com @8.8.8.8
done | grep real
```

A well-configured Pi-hole typically blocks 20-30% of DNS queries but has negligible performance impact. Misconfigured blocklists (too many rules) can add 50-100ms per query.

## Bypassing Advanced Tracker Evasion

Some trackers use sophisticated evasion techniques:

### Probabilistic Tracking

Trackers hash browser fingerprints without storing identifiable data. This is mathematically harder to block.

### First-Party Data Collection

Services embedded on the site (chat widgets, payment processors) can collect data directly. These cannot be blocked by DNS since they're on the same domain.

### Canvas Fingerprinting

JavaScript code accesses your graphics card information through Canvas API without making external requests. Block this through Content-Security-Policy:

```html
<!-- Restrict canvas access -->
<script>
HTMLCanvasElement.prototype.toDataURL = () => {
  throw new Error('Canvas fingerprinting blocked');
};
</script>
```

## Related Articles

- [How to Audit Your Browser Extensions for Privacy](/privacy-tools-guide/how-to-audit-your-browser-extensions-for-privacy-risks/)
- [How To Detect And Block Hidden Third Party Trackers On](/privacy-tools-guide/how-to-detect-and-block-hidden-third-party-trackers-on-websi/)
- [Privacy Implications of Browser Extensions](/privacy-tools-guide/privacy-implications-browser-extensions/)
- [Best Privacy Browser Extensions Ranked by Performance](/privacy-tools-guide/best-privacy-browser-extensions-ranked-by-performance-impact/)
- [How to Audit Chrome Extensions for Privacy](/privacy-tools-guide/audit-chrome-extensions-privacy-guide/)
- [Cursor AI Privacy Mode How to Use AI Features](https://theluckystrike.github.io/ai-tools-compared/cursor-ai-privacy-mode-how-to-use-ai-features-without-sendin/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
