---
layout: default
title: "How to Harden Firefox for Privacy (2026)"
description: "Firefox privacy hardening guide: about:config tweaks, uBlock Origin setup, container tabs, arkenfox user.js, DNS settings, tracking prevention."
author: "Privacy Tools Guide"
date: 2026-03-22
updated: 2026-03-22
reviewed: true
score: 9
voice-checked: true
intent-checked: true
slug: how-to-harden-firefox-for-privacy-2026
tags: [privacy-tools-guide, firefox, privacy, security, browser-hardening, tracking]
permalink: /how-to-harden-firefox-for-privacy-2026/
---

{% raw %}

Why Harden Firefox?

Firefox is the privacy browser. Unlike Chromium-based browsers (Chrome, Edge, Brave), Firefox isn't developed by an advertising company. But Firefox defaults are still weak. Tracking pixels, third-party cookies, DNS leaks, and browser fingerprinting work out of the box.

Real hardening requires about:config tweaks, uBlock Origin, container tabs, and DNS over HTTPS. This guide shows how to go from "Firefox default" to "corporate security standard."

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1: Phase 1: Built-in Privacy Settings

Open Firefox Preferences (about:preferences).

Permissions

Location: Preferences > Privacy & Security > Permissions

- Location: Deny unless you explicitly allow per-site
- Camera: Deny unless you explicitly allow per-site
- Microphone: Deny unless you explicitly allow per-site
- Notifications: Deny unless you explicitly allow per-site
- Autoplay: Block Audio and Video

Set autoplay to "Block Audio and Video". Websites autoplay videos and audio ads otherwise. Denying autoplay is not paranoia; it's refusing to download unnecessary bandwidth and data.

Tracking and Fingerprinting

Location: Preferences > Privacy & Security > Enhanced Tracking Protection

Select: Strict

Strict mode blocks:
- Trackers & ads: Known tracking scripts (Google Analytics, Facebook Pixel, etc.)
- Cryptominers: Scripts that hijack your CPU
- Fingerprinters: Scripts that identify you by browser characteristics (fonts, extensions, screen resolution)

Fingerprinting is underrated. If a tracker script learns your screen resolution, installed fonts, and browser version, it can identify you across websites (even without cookies). Blocking fingerprinters prevents this.

Cookies

Location: Preferences > Privacy & Security > Cookies and Site Data

Select: Cookies and site data for visited sites only

Or more paranoid: Cookies and site data (delete when Firefox is closed)

The first option accepts cookies from sites you visit but deletes them on browser close. Third-party cookies (ads, trackers) are blocked by default.

The second option never stores cookies. Extreme but effective. You'll need to re-login to every site daily.

I recommend "visited sites only." It's practical but still secure.

Check: Delete cookies and site data when Firefox is closed - Yes

Check: Send websites a "Do Not Track" signal - Yes

DNT is ignored by most sites, but it costs nothing and some privacy-conscious sites honor it.

Step 2: Phase 2: DNS over HTTPS

DNS leaks expose your browsing history. When you visit example.com, your ISP's DNS server logs the query. They know you visited example.com (even if traffic is HTTPS-encrypted).

Firefox supports DNS over HTTPS (DoH). Your DNS queries are encrypted and sent to a trusted provider instead of your ISP.

Location: Preferences > Privacy & Security > DNS over HTTPS

Select: Enabled (Max Protection)

This uses Firefox's preferred DoH provider (Cloudflare by default).

Alternative providers:

If you distrust Cloudflare (they're in the US, subject to government requests):

- Quad9 (quad9.net): Blocks known malware/phishing domains in addition to DoH
- Mullvad (mullvad.net): Swedish provider, no logs policy, owned by privacy-focused Mullvad VPN company
- NextDNS (nextdns.io): Offer granular blocking (ads, adult, malware)

To select a custom provider, choose "Max Protection" and configure a specific IP address.

For most users, Cloudflare's DoH is fine. They explicitly log nothing.

Step 3: Phase 3: about:config Tweaks

This is where real hardening happens. about:config is Firefox's deep settings.

about:config has 2000+ settings. Changing wrong ones breaks Firefox. Below are battle-tested hardening settings.

Open about:config (paste in address bar, confirm warning).

Essential Privacy Tweaks

| Setting | Value | Why |
|---------|-------|-----|
| `privacy.trackingprotection.enabled` | true | Enable tracking protection (redundant if Strict mode enabled, but explicit) |
| `privacy.trackingprotection.socialtracking.enabled` | true | Block social media trackers (Facebook, Twitter buttons) |
| `privacy.firstparty.isolate` | true | Isolate cookies by first-party site. Prevents cross-site tracking. |
| `privacy.resistFingerprinting` | true | Resist browser fingerprinting. Spoof screen resolution, fonts, timezone. |
| `network.cookie.cookieBehavior` | 4 | Only allow first-party cookies. Block all third-party. |
| `network.dns.disablePrefetch` | true | Don't pre-resolve DNS for links you hover over. |
| `network.prefetch-next` | false | Don't prefetch pages Firefox predicts you'll visit. |
| `dom.disable_beforeunload` | true | Prevent pages from blocking navigation with "unsaved changes" popups. |
| `browser.sessionstore.max_tabs_undo` | 3 | Limit undo/restore history to 3 tabs (privacy: don't leak closed tab URLs). |

Most critical: `privacy.firstparty.isolate` and `network.cookie.cookieBehavior = 4`.

First-party isolation means each website gets a separate cookie jar. If you visit both google.com and facebook.com, they don't share cookies. Prevents cross-site tracking.

Security Tweaks

| Setting | Value | Why |
|---------|-------|-----|
| `browser.cache.disk.enable` | false | Disable disk cache. Use RAM only. On reboot, no cached pages remain. |
| `browser.sessionstore.privacy_level` | 2 | Never save closed tabs/windows (restore history is a privacy leak). |
| `extensions.activeThemeID` | firefox-compact-dark | Use dark mode (minor privacy benefit: prevents theme ID fingerprinting). |
| `browser.startup.homepage_override.mstone` | ignore | Suppress Firefox "what's new" pages on update. |
| `datareporting.policy.dataSubmissionPolicyAcceptedVersion` | 2 | Acknowledge data policy (removes nagging) |

WebRTC Leak Prevention

WebRTC (peer-to-peer communication) can leak your real IP even behind a VPN.

| Setting | Value | Why |
|---------|-------|-----|
| `media.peerconnection.enabled` | false | Disable WebRTC (breaks some sites, but prevents IP leak) |

Or, less extreme:

| Setting | Value | Why |
|---------|-------|-----|
| `media.peerconnection.ice.default_address_only` | true | Only use non-public IPs for WebRTC |

The second option allows WebRTC (for sites that need it) but prevents your real IP from leaking.

Telemetry Disabling

Firefox collects usage data (what features you use, crash reports). Disable if paranoid:

| Setting | Value | Why |
|---------|-------|-----|
| `datareporting.healthreport.uploadEnabled` | false | Don't send health reports to Mozilla |
| `toolkit.telemetry.enabled` | false | Disable telemetry |
| `toolkit.telemetry.archive.enabled` | false | Don't archive telemetry locally |
| `experiment.enabled` | false | Don't participate in experiments |

These are optional. Mozilla's telemetry is privacy-respecting (no personally identifying information). But if you want zero data sharing, disable them.

Step 4: Phase 4: Extensions

uBlock Origin (Essential)

uBlock Origin is the gold-standard ad and tracker blocker. Install from addons.mozilla.org.

Open uBlock Origin settings (icon in toolbar, click gear):

My Filters tab:
Add custom filters:
```
! Block banner cookies
www.example.com##.cookie-consent-banner
! Block YouTube shorts sidebar
youtube.com##ytd-reel-shelf-renderer
```

Custom filters block specific elements on specific sites. You can block cookie banners, video players, sidebars, etc.

Filter Lists tab:

Enable:
- uBlock filters: your basics (ads, trackers)
- uBlock filters - Privacy: privacy-specific rules
- uBlock filters - Badware risks: malware/phishing domains
- EasyList: ads
- EasyList Privacy: trackers
- Fanboy's Annoyance List: cookie banners, popups

Don't enable too many lists. Each filter list is an HTTP request on startup. 5-7 lists is optimal.

Privacy Badger (Supplementary)

Privacy Badger detects trackers that uBlock misses. It learns over time.

Install from addons.mozilla.org. No configuration needed. Leave it on.

Difference from uBlock: Privacy Badger uses heuristics (behavioral analysis). If a script loads on 20+ different sites, it's probably a tracker. uBlock uses known filter lists.

Together, they cover each other's blind spots.

HTTPS Everywhere

HTTPS Everywhere forces HTTPS even if a site defaults to HTTP.

This prevents ISP/network snooping of page content. Combined with DoH, your entire browsing is encrypted end-to-end.

Install from addons.mozilla.org. No configuration needed.

Step 5: Phase 5: Container Tabs (Multi-Account Containers)

Container Tabs isolate cookies by container. Each container is a separate browsing context.

Open gmail.com in the "Work" container. Your work Google account logs in. Open gmail.com in the "Personal" container. Your personal Google account logs in. Same site, different logins, no mixing.

This prevents cross-site tracking and enables multi-account usage.

Install "Firefox Multi-Account Containers" from addons.mozilla.org (official Mozilla extension).

Setup containers:

Click container icon (left side of address bar). Click "Manage Containers."

Create:
- Work: Color blue, icon briefcase
- Personal: Color green, icon circle
- Banking: Color red, icon lock (never log out, separate from browsing)
- Shopping: Color yellow, icon star
- Facebook: Color purple, icon (for Facebook specifically)

Assign automatic containers:

Click container menu > Manage Containers > Assign Websites.

Add:
- `gmail.com` → Work
- `facebook.com` → Facebook
- `banking.example.com` → Banking

When you visit these sites, they automatically open in the assigned container. Cookies don't leak between containers.

Workflow:
1. Visit work site in Work container
2. Visit personal site in Personal container
3. Visit banking site in Banking container
4. Work and Personal sites can't see each other's cookies

This is more powerful than incognito mode. Incognito is temporary (cleared on close). Containers are persistent per-container.

Step 6: Phase 6: Arkenfox user.js (Advanced)

Arkenfox is a Firefox hardening project. They maintain a `user.js` file with 500+ security settings.

What it does:
- Disables all telemetry and data collection
- Hardens fingerprinting resistance
- Enables privacy-first defaults
- Disables unnecessary features (WebRTC, Beacon API, etc.)

Install:

1. Download user.js from github.com/arkenfox/user.js
2. Place in Firefox profile directory:
 - Windows: `C:\Users\[username]\AppData\Roaming\Mozilla\Firefox\Profiles\[profile.default]\`
 - macOS: `~/Library/Application Support/Firefox/Profiles/[profile.default]/`
 - Linux: `~/.mozilla/firefox/[profile.default]/`
3. Restart Firefox

Arkenfox rewrites your about:config. On restart, all 500+ settings are applied.

Caveats:

Arkenfox is aggressive. Some sites break (banking logins fail, some JS doesn't work). You must maintain an overrides file:

Create `user-overrides.js` in the same profile directory with site-specific exceptions:

```javascript
// Allow Gmail to use Beacon API
user_pref("beacon.enabled", true);  // Default: false

// Allow Netflix to play video
user_pref("media.eme.enabled", true);  // Default: false (disables DRM)
```

Arkenfox loads user.js first, then user-overrides.js (overrides take precedence).

Is Arkenfox necessary?

Not for most users. The about:config tweaks in Phase 3 cover 95% of hardening. Arkenfox is for users who want maximum hardening and don't mind breakage.

If you use online banking, shopping, streaming, Arkenfox will break things. The Phase 3 tweaks are more practical.

Step 7: Phase 7: Verification and Testing

After hardening, verify your setup works:

Check Tracking Protection

Visit BraveHQ's "Block trackers" test (test-page.bravesoftware.com). Your tracker count should be >50 blocked.

Check Fingerprinting Protection

Visit coveryourtracks.eff.org. You should see "Your browser is harder to fingerprint."

Check DNS Leak

Visit dnsleaktest.com. All IPs should be from your DoH provider (Cloudflare, Mullvad, etc.), not your ISP.

Check WebRTC Leak

Visit ipleak.net. Your real IP should NOT be exposed.

Test Site Functionality

Visit a few regular sites (Gmail, Twitter, YouTube). If they work, your hardening is compatible.

Step 8: Common Breakage and Fixes

Issue: Site won't load

Often caused by blocking JavaScript or disabling a feature the site needs.

Fix: Temporarily add site to uBlock's whitelist (click uBlock icon, press spacebar). Refresh. If it loads, the site needs ads/trackers.

Add to uBlock custom filters:
```
! Allow example.com to load
@@||example.com^
```

Issue: Login fails

Often caused by first-party isolation or strict cookie settings.

Fix: Disable first-party isolation for that site:

In about:config, add to `privacy.firstparty.isolate.restrict_opener_access` an exception (advanced).

Or temporarily disable `privacy.firstparty.isolate`, test, re-enable.

Issue: Video won't play

Often caused by disabling WebRTC or DRM (Encrypted Media Extensions).

Fix: Check if site uses DRM (Netflix, Disney+, Amazon Prime). Temporarily enable `media.eme.enabled`. Refresh.

Step 9: Recommended Setup (Practical)

For most users, this setup balances privacy and usability:

1. Preferences:
 - Enhanced Tracking Protection: Strict
 - Cookies: Visited sites only, delete on close
 - Location/Camera/Microphone: Deny unless you allow
 - DoH: Enabled (Cloudflare)

2. about:config:
 - `privacy.firstparty.isolate = true`
 - `network.cookie.cookieBehavior = 4`
 - `privacy.resistFingerprinting = true`
 - `media.peerconnection.ice.default_address_only = true`

3. Extensions:
 - uBlock Origin (essential)
 - Privacy Badger (supplementary)
 - Firefox Multi-Account Containers (for multi-account)
 - HTTPS Everywhere (optional, most sites are HTTPS by default now)

4. Don't:
 - Install Arkenfox (too aggressive for practical use)
 - Disable JavaScript (breaks too many sites)
 - Disable all cookies (you'll log out constantly)

This setup achieves:
- Blocks 99% of trackers
- Prevents fingerprinting
- Encrypts DNS
- Isolates cookies by site
- Enables multi-account browsing

Setup time: 30 minutes. Ongoing maintenance: 0 hours.

Step 10: Desktop vs Mobile

Firefox Desktop (above) supports all tweaks.

Firefox Mobile (Android) supports:
- Enhanced Tracking Protection
- DoH
- uBlock Origin (addon support)
- Multi-Account Containers (addon support)

Setup is simpler but less powerful. Android Firefox still leaks some data.

For maximum mobile privacy, consider:
- Android: Firefox + uBlock Origin + Mullvad VPN
- iOS: Safari (built-in privacy features) or Firefox (limited addon support)

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to harden firefox for privacy (2026)?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Best Accessible Privacy Extension for Firefox That Does Not](/best-accessible-privacy-extension-for-firefox-that-does-not-/)
- [Firefox Privacy Add-ons Essential List 2026: Complete Guide](/firefox-privacy-add-ons-essential-list-2026/)
- [Firefox Privacy Settings Guide 2026](/firefox-privacy-settings-guide-2026/)
- [Configure Firefox for Maximum Privacy Without Breaking](/how-to-configure-firefox-maximum-privacy-without-breaking-sites/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
