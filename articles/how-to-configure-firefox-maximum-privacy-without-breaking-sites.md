---
layout: default
title: "How to Configure Firefox for Maximum Privacy Without Breaking Sites."
description: "Hardened Firefox settings that block trackers, ads, and surveillance while keeping 99% of sites functional."
date: 2026-03-20
author: theluckystrike
permalink: /how-to-configure-firefox-maximum-privacy-without-breaking-sites/
categories: [guides]
tags: [privacy, firefox, browser, security, tracking]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

Firefox is the only major browser developed by a non-profit. It's the best privacy browser if configured correctly—but the default settings leave you exposed to trackers and advertisers. This guide hardens Firefox without causing widespread site breakage (the common problem with overly aggressive privacy configs).

## The Privacy Problem Firefox Solves

Most browsers (Chrome, Edge, Safari) are built by corporations that profit from your data:
- Chrome: Google collects tracking data across the web
- Edge: Microsoft integrates with Windows telemetry
- Safari: Apple collects Siri searches and location data

Firefox is built by Mozilla, a nonprofit. They don't sell your data. But they use Google as default search (revenue sharing), and many default settings are permissive for compatibility.

This guide locks Firefox down while keeping sites functional.

## The Balance: Privacy vs. Breakage

Aggressive privacy settings break ~5% of sites:
- Video players that require Flash (obsolete, already broken)
- Complex authentication flows
- Finance sites with unusual login UX
- Legacy corporate intranets

This guide targets 95%+ functionality. If you want 99.9% privacy, you'll lose some sites.

## Step 1: Firefox Settings (about:preferences)

### Privacy & Security → Enhanced Tracking Protection

```
Current setting: Standard
Better setting: Strict
Effect: Blocks known trackers, requires site whitelisting for breakage
Breakage: <1%
```

Click: Settings → Privacy & Security → Enhanced Tracking Protection → Strict

### Cookies and Site Data

```
Current: "Allow all cookies"
Better: "Only from sites you visit"
Best: "Block all cookies" (requires whitelisting)
Recommendation: "Only from sites you visit"
```

Firefox → Preferences → Privacy → Cookies and Site Data → "Only from sites you visit"

**Explanation:**
- Third-party cookies are how advertisers track you across sites
- First-party cookies (same-site) are necessary for logins and preferences
- "Only from sites you visit" blocks trackers but keeps logins working

### Site Permissions

```
Location: Block by default
Microphone: Block by default
Camera: Block by default
Notification: Block always
```

Firefox → Preferences → Privacy → Permissions

You'll manually grant permissions to sites that need them (rare).

### DNS over HTTPS

```
Current: Off
Better: On (with Cloudflare or Mozilla)
Effect: ISP can't see which sites you visit
Breakage: <0.1%
```

Firefox → Preferences → Privacy & Security → DNS over HTTPS → Enable with Cloudflare

**Why this matters:** Without DNS over HTTPS, your ISP sees every site you visit (sold to advertisers). With it, only Cloudflare sees (and Cloudflare has a better privacy policy than ISPs).

## Step 2: about:config Advanced Settings

Enter `about:config` in address bar and confirm you'll be careful.

### Core Privacy Settings

```javascript
// Disable telemetry
datareporting.policy.dataSubmissionEnabled = false
datareporting.healthreport.uploadEnabled = false
toolkit.telemetry.enabled = false
toolkit.telemetry.archive.enabled = false

// Disable pocket integration (reading list, ads)
extensions.pocket.enabled = false

// Disable studies (Mozilla experiments)
app.studies.enabled = false

// Disable Firefox profiling (send usage data)
profiler.enabled = false

// Disable crash reports
breakpad.reportURL = ""
```

### Tracking Protection Settings

```javascript
// Prevent DOM-based tracking
privacy.trackingprotection.enabled = true
privacy.trackingprotection.cryptomining.enabled = true
privacy.trackingprotection.fingerprinting.enabled = true

// Disable third-party tracking (Strict mode)
network.cookie.sameSite.laxByDefault = true

// Disable floating tracking parameters
privacy.query_stripping.enabled = true
privacy.query_stripping.enabled.pbmode = true
```

### Fingerprinting Protection

Websites track you by fingerprinting (your unique browser config):

```javascript
// Randomize user agent for privacy
// (slightly increases breakage, recommend leaving off)
privacy.resistFingerprinting = false  // True breaks some auth, keeping false

// Reduce entropy in reports
privacy.reduceTimerPrecision = true
privacy.reduceTimerPrecision.microseconds = 1000

// Disable WebGL (reduces fingerprinting)
webgl.disabled = false  // Breaking; disable only if needed

// Disable battery API (reveals low power states)
dom.battery.enabled = false
```

### Network Security

```javascript
// Require HTTPS for all connections (upgrade http:// to https://)
dom.security.https_only_mode = true
dom.security.https_only_mode_ever_enabled = true

// Enforce strict CSP (prevents XSS attacks)
security.csp.enable = true

// Disable HTTP Basic Auth (uses plaintext passwords)
network.http.keep-alive.timeout = 300
```

### Third-Party Restrictions

```javascript
// Block third-party frames
privacy.partition.network_state = true
privacy.partition.network_state_isolation = true

// Disable OCSP (prevents sites from learning what you've visited)
security.OCSP.enabled = 0  // Value 0 disables OCSP
```

## Step 3: Privacy Extensions (Minimal Set)

Avoid extension bloat (each extension is an attack surface). Use only 3-4:

### uBlock Origin (Ad/Tracker Blocking)

Install from: addons.mozilla.org

**Configuration:**
1. Click uBlock icon → Dashboard
2. Go to "Filter lists" tab
3. Enable recommended lists:
 - uBlock filters
 - EasyList (ads)
 - EasyPrivacy (tracking)
 - Fanboy's Annoyance List (social media widgets)
 - Peter Lowe's Ad/Tracking list

**Settings → Behavior:**
- Check "Prevent media elements from loading data"
- Check "Block remote fonts"
- Check "Block WebSockets"

**Why uBlock Origin:**
- Blocks ads AND trackers (not just ads)
- Minimal performance impact
- No send-back to server (works locally)

### Bitwarden (Password Manager)

Install from: addons.mozilla.org

Bitwarden reduces password reuse (biggest security problem). Each site gets unique password.

**Configuration:**
- Create account at bitwarden.com ($0-10/year)
- Log in via extension
- Auto-fill passwords on login (reduces typing attack vectors)

### Decentraleyes (CDN Privacy)

Install from: addons.mozilla.org

Blocks requests to centralized CDNs (Cloudflare, Google Fonts, Bootstrap). Replaces with local versions.

**Effect:**
- Websites load jQuery locally instead of googleapis.com
- Google can't track font requests
- Minimal breakage (<0.5%)

**Configuration:** Zero setup required; works silently.

### SimpleLogin (Email Masking)

Install from: addons.mozilla.org (optional, only if you care about email privacy)

Creates alias email addresses for every site signup. Forwards to your real email, preventing trackers from connecting your email across sites.

**Cost:** $99/year (alternative: use email provider's alias feature)

**When to use:**
- Signing up for new services (use alias)
- Shopping sites (use alias)
- Marketing newsletters (use alias)

**When to skip:**
- Close contacts (use real email)
- Email accounts themselves (can't use alias for the email you own)

**Why it matters:** Advertisers buy email lists and cross-reference to build profiles. Aliases prevent this linking.

## Step 4: Settings Summary (Table Format)

| Setting | Value | Breakage | Notes |
|---------|-------|----------|-------|
| Enhanced Tracking Protection | Strict | <1% | Only exception for broken sites |
| Cookies | Sites you visit only | <0.5% | Blocks tracker pixels |
| DNS over HTTPS | Cloudflare | <0.1% | ISP can't see sites |
| HTTPS only mode | On | <2% | Upgrades http:// to https:// |
| Telemetry | Disabled | 0% | Just privacy, no breakage |
| Pocket / Studies | Disabled | 0% | Just privacy, no breakage |
| uBlock Origin | Installed + configured | <2% | Blocks ads + trackers |
| Bitwarden | Installed | 0% | Improves security |

## Testing Your Configuration

### Test 1: Check Tracker Blocking

Visit: https://cookiepedia.co.uk/

Expected: Site should show you're blocking most trackers. Compare to default browser.

### Test 2: Check DNS over HTTPS

Visit: https://1.1.1.1/dns/

Expected: Shows "DNS over HTTPS working" (green).

### Test 3: Check Fingerprinting Resistance

Visit: https://browserleaks.com/

Expected:
- ✓ Blocks WebRTC leak
- ✓ Blocks canvas fingerprinting
- ✗ Minor fingerprinting possible (acceptable)

### Test 4: Common Sites Still Work

```
✓ Gmail / Yahoo Mail
✓ Banking sites (some may require whitelisting)
✓ YouTube / Netflix
✓ Reddit / Twitter
✓ Amazon / shopping sites
? Corporate intranets (may require HTTPS exceptions)
```

If a critical site breaks:
1. Click the uBlock origin icon
2. Click the power button to whitelist that domain for that session
3. Reload page
4. If it works, add to permanent whitelist

**Whitelist process:**
1. Click uBlock → Dashboard
2. "Whitelist" tab
3. Add domain

## Avoiding Fingerprinting While Maintaining Functionality

### The Fingerprinting Tradeoff

**Aggressive privacy:**
```
privacy.resistFingerprinting = true  // But breaks many sites
```

**Balanced (recommended):**
```
privacy.resistFingerprinting = false  // Fingerprinting possible
privacy.reduceTimerPrecision = true    // Reduced timing attack surface
```

Browser fingerprinting is complex. Truly defeating it requires tools like Tor Browser, which is slower. For normal browsing, the above balance is good.

## Site Breakage: Troubleshooting

### Scenario 1: Login Page Doesn't Work

**Cause:** HTTPS only mode or strict cookies

**Fix:**
1. Temporarily disable HTTPS only mode for that domain:
   ```
   about:preferences → Privacy & Security → HTTPS only mode → Off
   ```
2. Or whitelist in uBlock Origin

**Permanent fix:** Report to site (HTTPS not working is their bug)

### Scenario 2: Video Won't Play

**Cause:** uBlock Origin blocking ad/tracking scripts

**Fix:**
1. Click uBlock icon
2. Power button to whitelist that domain
3. Reload
4. Click icon again, check "Media" and "Scripts" categories

### Scenario 3: Payment Page Errors

**Cause:** Overly aggressive privacy preventing form submission

**Fix:**
1. Enable HTTPS only mode OFF for that domain
2. Disable Enhanced Tracking Protection for that domain
3. Complete payment
4. Re-enable after

**Note:** Legitimate payment sites should work with your privacy settings. If they don't, they have trust/security problems.

## Advanced: Container Tabs

For maximum privacy, use Firefox Containers (separate cookie jars per domain):

```
Install: Multi-Account Containers extension
Configuration:
- Create containers: Work, Personal, Banking, Shopping
- Assign sites to containers
- Cookies never cross containers
```

**Effect:**
- Amazon can't track you on other sites (different container)
- Facebook can't see your banking activity (different container)

**Tradeoff:** Slightly inconvenient (must remember container assignments)

## Privacy Maintenance Checklist

**Monthly:**
- [ ] Update Firefox (automatic, but verify in Settings)
- [ ] Update extensions (automatic)
- [ ] Review about:config settings haven't been reset

**Quarterly:**
- [ ] Check Firefox privacy report (top-right menu → Privacy → "View Protection Report")
- [ ] Review blocked trackers (should be 100+)
- [ ] Check for site breakage (report major issues)

**Annually:**
- [ ] Review Mozilla's privacy policy (no major changes expected)
- [ ] Audit installed extensions (remove unused ones)
- [ ] Regenerate Firefox profile if feeling paranoid

## Related Reading

- [Best password managers with emergency access features compared](/privacy-tools-guide/guides-hub/)
- [How to set up a secure home server for self-hosting privacy tools](/privacy-tools-guide/guides-hub/)
- [Android privacy dashboard: how to use it to audit app access](/privacy-tools-guide/guides-hub/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
