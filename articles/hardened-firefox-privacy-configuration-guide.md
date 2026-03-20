---
layout: default
title: "Hardened Firefox Privacy Configuration Guide"
description: "Complete Firefox hardening guide: about:config tweaks, essential extensions, arkenfox user.js profiles, and step-by-step configuration."
date: 2026-03-20
author: "Privacy Tools Guide"
permalink: /hardened-firefox-privacy-configuration/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of, privacy]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Firefox is the only major browser where privacy hardening is practical and effective. Unlike Chrome (owned by ad company Google) or Safari (locked to Apple ecosystem), Firefox is:

1. **Open-source**: Community audits the code
2. **Private by default**: No forced Google sync, no ads
3. **Configurable**: Deep about:config settings available
4. **Customizable**: Community-maintained hardening profiles (arkenfox)

But default Firefox isn't private. It collects telemetry, allows fingerprinting, doesn't block third-party cookies, and leaks your real IP during WebRTC calls.

This guide hardens Firefox for maximum privacy. It covers:

1. **about:config tweaks**: 40+ settings that eliminate tracking
2. **Essential extensions**: uBlock Origin, NoScript, Privacy Badger
3. **arkenfox user.js**: Community hardening profile (copy-paste configuration)
4. **DNS and proxy**: Encrypted DNS, VPN integration
5. **Fingerprinting resistance**: Browser identification prevention

## Privacy Baseline: What Firefox Leaks by Default

Test your default Firefox at [browserleaks.com](https://browserleaks.com):

Default Firefox reveals:
- **Canvas fingerprint**: Unique identifier based on your GPU/fonts
- **WebGL fingerprint**: GPU model, rendering engine
- **IP geolocation**: Exact location (even with VPN, if WebRTC leaks)
- **User-Agent**: OS, browser version
- **Referrer headers**: Sites you came from
- **Third-party cookies**: Trackers follow you across web

## Part 1: Essential about:config Settings

Open Firefox. Type `about:config` in address bar. Press Enter.

You'll see a warning. Click "Accept the Risk and Continue."

Now search for settings and change them. (Use Ctrl+F to search.)

### Privacy Core Settings

**Disable telemetry:**

```
datareporting.healthreport.uploadEnabled = false
datareporting.policy.dataSubmissionEnabled = false
toolkit.telemetry.archive.enabled = false
toolkit.telemetry.enabled = false
toolkit.telemetry.reportingpolicy.firstRun = false
```

**Disable Firefox studies (experiments):**

```
app.shield.optoutstudies.enabled = false
app.normandy.enabled = false
```

**Disable pocket (Mozilla's content recommendation):**

```
extensions.pocket.enabled = false
```

**Disable pocket's background features:**

```
browser.newtab.preload = false
```

**Disable new tab suggestions (freatures data collection):**

```
browser.newtabpage.activity-stream.feeds.section.topstories = false
browser.newtabpage.activity-stream.feeds.snippets = false
```

### Cookie and Tracking Settings

**Block third-party cookies:**

```
network.cookie.cookieBehavior = 4
```

(Values: 0=allow all, 1=third-party only, 2=none, 3=no tracking cookies, 4=third-party except visited)

**Set to value 4**: Block third-party cookies except those on sites you've visited.

**Disable cookie tracking in private mode:**

```
network.cookie.privacyLevel = 2
```

### Fingerprinting Resistance

**Enable "Resist Fingerprinting" (built-in feature):**

```
privacy.resistFingerprinting = true
```

This:
- Randomizes canvas fingerprints
- Rounds time precision (prevents timing attacks)
- Disables WebGL fingerprinting
- Standardizes user-agent across sites

Downside: Some websites break (especially financial sites). Accept this tradeoff for privacy.

**Disable WebGL (GPU fingerprinting):**

```
webgl.disabled = true
```

**Disable WebGL2:**

```
webgl2.disabled = true
```

**Spoof timezone to UTC (standard):**

```
privacy.spoof_english = 2
```

### Network and Referrer

**Disable referrer leaking (strict):**

```
network.http.referer.XOriginPolicy = 2
```

(0=always send, 1=same-site only, 2=same-host only)

**Trim referrer to origin only:**

```
network.http.referer.XOriginTrimmingPolicy = 2
```

**Disable IPv6 (reduces fingerprinting surface):**

```
network.dns.disableIPv6 = true
```

### WebRTC IP Leak Prevention

WebRTC (video calling API) leaks your real IP even through VPN. Block it:

```
media.peerconnection.enabled = false
```

This breaks video calling in browser, but prevents IP leak. For video calls, use Jitsi or Signal instead.

Alternatively, if you need WebRTC:

```
media.peerconnection.ice.no_host = true
media.peerconnection.ice.default_address_only = true
```

(Leak your local network IP only, not ISP IP.)

### HTTPS-Only Mode

**Force HTTPS everywhere:**

```
dom.security.https_only_mode = true
dom.security.https_only_mode_ever_enabled = true
```

### DOM Storage and Cache

**Disable DOM storage (used by trackers):**

```
dom.storage.enabled = false
```

**Disable service workers (can enable offline tracking):**

```
dom.serviceWorkers.enabled = false
```

**Clear cache on exit:**

```
privacy.sanitize.sanitizeOnShutdown = true
privacy.clearOnShutdown.cache = true
privacy.clearOnShutdown.cookies = true
privacy.clearOnShutdown.history = true
```

## Part 2: Essential Extensions

Firefox extensions are the second layer of privacy hardening.

### 1. uBlock Origin (Essential)

**What it does**: Ad and tracker blocker. Blocks ads, malware sites, and tracking domains using curated filter lists.

**Installation**:

1. Go to [addons.mozilla.org/extensions/ublock-origin](https://addons.mozilla.org/en-US/firefox/addon/ublock-origin/)
2. Click "Add to Firefox"
3. Open dashboard (click uBlock icon → Settings gear)

**Configuration**:

Go to **Dashboard → Filter lists**. Enable:

- uBlock filters (enabled by default)
- EasyList (ad blocking)
- EasyPrivacy (tracking blocking)
- Malware Domains
- Peter Lowe's Ad/tracking server list
- MVPS HOSTS
- Add-on: Adguard Base filter

**Advanced settings** (click gear icon in dashboard):

```
Enable "I am an advanced user"
```

In filter editor, paste:

```
! Block WebRTC IP leak
no-strict3p-exception: youtube.com|youtu.be

! Block pixel trackers
||doubleclick.net^
||google-analytics.com^
||facebook.com/tr^
```

### 2. Privacy Badger (Essential)

**What it does**: Auto-learns which trackers follow you and blocks them.

**Installation**: [addons.mozilla.org/privacy-badger](https://addons.mozilla.org/en-US/firefox/addon/privacy-badger17/)

**Configuration**:

- Click extension icon
- "Open Settings"
- Enable: "Block tracking ads", "Enable learning mode"

Privacy Badger learns from your browsing. Over 1-2 weeks, it identifies trackers and blocks them. Zero configuration needed.

### 3. Decentraleyes (Recommended)

**What it does**: Prevents Content Delivery Networks (CDNs) from tracking you by hosting common libraries locally.

**Example**: Facebook Pixel is loaded on thousands of websites via Cloudflare CDN. CDN logs your IP, identifies you across sites. Decentraleyes serves Pixel locally instead.

**Installation**: [addons.mozilla.org/decentraleyes](https://addons.mozilla.org/en-US/firefox/addon/decentraleyes/)

**Configuration**: Default settings are fine. No setup needed.

### 4. NoScript (Advanced Users)

**What it does**: Blocks JavaScript by default. You whitelist trusted sites.

**Installation**: [addons.mozilla.org/noscript](https://addons.mozilla.org/en-US/firefox/addon/noscript/)

**Warning**: Many websites require JavaScript. This is a power-user tool.

**Configuration**:

- Default mode: Block all scripts
- When a site breaks, click NoScript icon
- Click "Temporarily Allow [domain]" to whitelist

**Example**:

```
Gmail.com → Allow (needed for email)
Random news site → Block (works without JS, just disables ads)
YouTube.com → Allow (video requires JS)
Tracker domain → Block (never whitelist)
```

### 5. HTTPS Everywhere (Fallback)

**What it does**: Forces HTTPS on websites that support it.

**Installation**: [addons.mozilla.org/https-everywhere](https://addons.mozilla.org/en-US/firefox/addon/https-everywhere/)

**Note**: Modern Firefox has HTTPS-Only mode (about:config setting above). This is redundant but doesn't hurt.

### Extensions to AVOID

- **Brave Search**: Proprietary, closed-source
- **Ghostery**: Corporate tracking company
- **Avast/AVG extensions**: Privacy-invasive VPN providers

## Part 3: arkenfox user.js

**arkenfox** is a community-maintained user.js file—a hardening profile that applies 100+ privacy and security settings at once.

### What is user.js?

A `user.js` file in your Firefox profile applies settings automatically on startup. Instead of manually setting 50+ about:config options, you copy one file.

### Installation

**Step 1: Find your Firefox profile folder**

**On macOS**:

```
~/Library/Application Support/Firefox/Profiles/RANDOMSTRING.default-release/
```

**On Linux**:

```
~/.mozilla/firefox/RANDOMSTRING.default-release/
```

**On Windows**:

```
%APPDATA%\Mozilla\Firefox\Profiles\RANDOMSTRING.default-release\
```

**Step 2: Download arkenfox**

Visit [github.com/arkenfox/user.js](https://github.com/arkenfox/user.js)

Click "Code" → "Download ZIP"

Unzip. Open the folder.

**Step 3: Copy user.js to Firefox profile**

Copy the `user.js` file from the downloaded arkenfox folder into your Firefox profile folder (found in Step 1).

**Step 4: Restart Firefox**

Close and reopen Firefox.

Settings from arkenfox apply automatically.

### What arkenfox Changes

arkenfox enables:

- **Telemetry disabled**: No data collection
- **Fingerprinting blocked**: Canvas, WebGL, user-agent randomized
- **Cookies strict**: Third-party blocked, strict SameSite
- **HTTPS-only**: All connections encrypted
- **WebRTC disabled**: No IP leak
- **DOM storage disabled**: No persistent tracking storage
- **Service workers disabled**: No offline tracking

### arkenfox Overrides (Overrides and Smoothing)

arkenfox is strict—it breaks some sites. You can override specific settings.

Create `user-overrides.js` in the same profile folder:

```javascript
// Allow YouTube to work properly
user_pref("media.mediasource.enabled", true);
user_pref("media.mediasource.webm.enabled", true);

// Allow some Google services
user_pref("network.cookie.cookieBehavior", 1);

// Allow Discord to work
user_pref("dom.serviceWorkers.enabled", true);
```

arkenfox loads `user.js` first, then `user-overrides.js` (overriding stricter settings).

### Testing arkenfox

After installation, test privacy at [browserleaks.com](https://browserleaks.com):

**Before arkenfox**:

```
Canvas fingerprint: Unique ID
WebGL fingerprint: Unique ID
IP address: Real IP
User-Agent: Windows 10, Firefox 124
Timezone: EST
```

**After arkenfox**:

```
Canvas fingerprint: Randomized (different each page load)
WebGL fingerprint: Disabled
IP address: No WebRTC leak (VPN IP if using VPN)
User-Agent: Standardized (all look identical)
Timezone: UTC
```

## Part 4: DNS Privacy

**Default DNS**: Your ISP's DNS server logs all domains you visit.

Example: You visit `medical-clinic.com` → ISP logs it → Can be subpoenaed in legal cases.

### Encrypted DNS (DoH/DoT)

Use **Encrypted DNS (DoH)** to hide DNS queries from ISP.

**In Firefox, about:config**:

```
network.trr.mode = 2
network.trr.uri = https://dns.nextdns.io/YOUR_NEXTDNS_ID
```

(Replace `YOUR_NEXTDNS_ID` with your NextDNS account ID.)

**Or use Cloudflare 1.1.1.1** (Privacy-focused DNS):

```
network.trr.uri = https://mozilla.cloudflare-dns.com/dns-query
```

### NextDNS (Recommended)

**Cost**: Free tier (300k queries/month); $1.99/month unlimited

**What it does**:

- Block ads, trackers, malware before they reach your device
- Encrypt DNS queries (ISP can't see domains)
- Privacy dashboard (see what got blocked)
- Works across all devices (phone, laptop, router)

**Setup**:

1. Create account at [nextdns.io](https://nextdns.io)
2. Go to "Configuration"
3. Enable: "Privacy", "Security", "Parental Controls"
4. Copy your NextDNS ID
5. In Firefox about:config, set:

```
network.trr.uri = https://dns.nextdns.io/YOUR_ID
network.trr.mode = 2
```

6. Test at [nextdns.io/dash](https://nextdns.io/dash) → see blocked queries

## Part 5: VPN Integration (Optional)

Firefox doesn't have built-in VPN. For IP privacy, use a VPN + Firefox together.

**Recommended VPNs**:

- **Mullvad**: $5/month, no logging, open-source
- **ProtonVPN**: $4.99/month, no logging, Swiss-based
- **IVPN**: $6/month, no logging, transparent logging policy

**Setup**:

1. Install VPN application
2. Enable "Always On VPN"
3. Firefox traffic routes through VPN automatically
4. Test IP at [ipleak.net](https://ipleak.net) → Should show VPN IP, not real IP

## Part 6: Hardened Profile Checklist

Use this checklist to verify your Firefox is hardened:

**about:config Settings**:

- [ ] `privacy.resistFingerprinting = true`
- [ ] `network.cookie.cookieBehavior = 4`
- [ ] `media.peerconnection.enabled = false` (or WebRTC leak prevention)
- [ ] `dom.security.https_only_mode = true`
- [ ] `datareporting.healthreport.uploadEnabled = false`
- [ ] `network.http.referer.XOriginPolicy = 2`

**Extensions**:

- [ ] uBlock Origin installed and configured
- [ ] Privacy Badger installed
- [ ] Decentraleyes installed
- [ ] NoScript installed (optional but recommended)

**arkenfox**:

- [ ] user.js installed in profile folder
- [ ] user-overrides.js created for broken sites
- [ ] Firefox restarted

**DNS**:

- [ ] Encrypted DNS enabled (NextDNS or Cloudflare)
- [ ] Test at [nextdns.io/dash](https://nextdns.io/dash)

**VPN** (optional):

- [ ] VPN installed and set to "Always On"
- [ ] IP test shows VPN IP, not real IP

**Verify Privacy**:

- [ ] Visit [browserleaks.com](https://browserleaks.com)
- [ ] Canvas fingerprint: Randomized
- [ ] WebGL: Disabled or randomized
- [ ] WebRTC leak: None

## Real-World Performance Impact

After hardening, expect:

- **Speed**: 5-10% slower (minor). Ad/tracker blocking saves time loading ads.
- **Breakage**: 10-15% of websites have issues
  - Example: Some banking sites require JavaScript
  - Solution: Whitelist in NoScript
- **Memory usage**: 10-15% higher (more extensions)

## Maintenance

**Monthly**:

- Check [github.com/arkenfox/user.js](https://github.com/arkenfox/user.js) for updates
- Download latest user.js, replace in profile folder

**Weekly**:

- Review [nextdns.io/dash](https://nextdns.io/dash) for blocked domains
- Adjust if legitimate sites blocked

## Limitations

Firefox hardening prevents *tracking*, but doesn't prevent:

- **Content identification**: Some sites can identify you by browsing behavior (e.g., "User read articles on machine learning" → Probably engineer)
- **IP-level tracking**: Your VPN provider still sees your IP (trust your VPN provider)
- **Account-based tracking**: If logged into Gmail/Facebook, Google/Facebook track you regardless
- **JavaScript exploits**: JavaScript can still extract information

## Recommendations

1. **Start small**: Set about:config privacy options first (easiest)
2. **Add extensions**: Install uBlock + Privacy Badger (no breaking changes)
3. **Deploy arkenfox**: Copy-paste user.js (most aggressive, will break some sites)
4. **Use DNS encryption**: NextDNS Free tier (no cost, high value)
5. **VPN optional**: Add VPN if ISP privacy is concern

**For average user**: Steps 1 + 2 + 4 = 80% privacy gain with minimal breakage.

**For privacy-focused user**: All steps = maximum privacy, accept breakage, whitelist sites as needed.

{% endraw %}

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

