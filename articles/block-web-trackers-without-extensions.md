---
layout: default
title: "How to Block Web Trackers Without Extensions"
description: "Use DNS-level blocking, browser settings, and system-wide hosts files to stop web trackers without installing browser extensions or ad blockers"
date: 2026-03-22
author: theluckystrike
permalink: /block-web-trackers-without-extensions/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Block Web Trackers Without Extensions

Browser extensions are the standard advice for blocking trackers, but they come with trade-offs: they can see everything you browse, they add attack surface, and some have had vulnerabilities. DNS-level blocking, system hosts files, and browser native settings catch most trackers without adding extensions to your browser.

---

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

Pi-hole blocks ~20-25% of all DNS queries on average home networks.

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

---

## Layer 4: Firefox Advanced Config

```
Firefox → about:config:
network.http.referer.XOriginPolicy = 2  # Don't send Referer cross-origin
dom.storage.enabled = false             # Breaks some sites but blocks storage tracking
```

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

---

## What This Approach Misses

DNS and hosts blocking doesn't cover:

- **First-party tracking**: Trackers hosted on the same domain as the site
- **CNAME cloaking**: Trackers that use the site's own subdomain as a CNAME alias
- **JS fingerprinting**: Runs entirely in the page, no outbound network call to block

For fingerprinting protection, Firefox with `privacy.resistFingerprinting = true` reduces the uniqueness of your browser signature by reporting standardized values for screen size, fonts, and timezone.

---

## Related Reading

- [Android Privacy Dashboard How to Use It](/privacy-tools-guide/android-privacy-dashboard-how-to-use-it/)
- [Audio Context Fingerprinting How Websites Use Sound API to Track](/privacy-tools-guide/audio-context-fingerprinting-how-websites-use-sound-api-trac/)
- [Android App Permissions Audit Guide 2026](/privacy-tools-guide/android-app-permissions-audit-guide-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
