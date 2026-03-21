---
layout: default
title: "Tor Browser Security Settings Configuration Guide"
description: "How to configure Tor Browser's security levels, circuit controls, JavaScript settings, and anti-fingerprinting features for different threat scenarios"
date: 2026-03-21
author: theluckystrike
permalink: /tor-browser-security-settings-guide/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Tor Browser works out of the box. Its default settings protect most users in most situations. But understanding what each setting does — and when to change it — lets you make informed decisions rather than clicking through defaults. This guide explains the security controls, when to use each level, and the common mistakes that undermine Tor Browser's anonymity.

## The Three Security Levels

Tor Browser's most important control is the Security Level, accessible from the shield icon next to the address bar, or from Settings → Privacy & Security → Security Level.

### Standard

JavaScript enabled. All browser features available. Recommended for: browsing news, Wikipedia, and general sites where functionality matters and de-anonymization risk is lower.

**What's on**: Full JavaScript, WebGL, video/audio codecs, all CSS features.

**Risk**: JavaScript is the most common vector for browser exploits and fingerprinting techniques that can de-anonymize Tor Browser users. A zero-day JavaScript exploit can potentially reveal your real IP or identity.

### Safer

Disables JavaScript on non-HTTPS sites. Disables some mathematical operations used for certain attacks. Disables audio and video content.

**What's on**: JavaScript only on HTTPS sites. HTML5 media requires click-to-play.

**Recommended for**: Most serious use cases — journalism, political activity in restrictive countries, general sensitive research. The trade-off in usability is modest; most important content is on HTTPS.

### Safest

JavaScript disabled everywhere. Only HTTPS content rendered. Plain HTML and CSS only.

**What's on**: Static content only. Many websites are partially or completely broken.

**Recommended for**: Maximum threat scenarios — accessing hidden services, communicating in high-risk environments, situations where a browser exploit could be life-threatening.

```
Standard   → general browsing, lower risk
Safer      → sensitive work, recommended default for most users
Safest     → maximum threat, activist/journalist in hostile environment
```

## Circuit Controls

Each tab in Tor Browser uses a circuit: a path through three Tor relays (guard → middle → exit). The exit relay is the one that communicates with the destination website.

### New Circuit for this Site

Click the lock icon next to the address bar → New Circuit for this Site.

This assigns a new exit relay for the current site. Your guard relay stays the same (this is by design — changing guards too often weakens anonymity). Use this when:
- A site is slow or broken (try a different exit node)
- You want to change the apparent country of origin for the next request
- You're starting a fresh session with a specific site

### New Identity

Click the menu (hamburger) → New Identity, or press Ctrl+Shift+U.

This is the nuclear option: closes all tabs, clears all state (cookies, session storage, cache), and establishes entirely new circuits including a new guard relay position.

Use this when you want to break the connection between your current browsing session and any future browsing. This is the correct way to switch between "personas" in Tor Browser.

Do not use New Identity frequently during a single sensitive task — the act of requesting new circuits repeatedly can itself be a pattern observable to relay operators.

## The No-Extensions Rule

Do not install extensions in Tor Browser. This is one of the most critical rules.

Tor Browser's anonymity depends on all users presenting an identical fingerprint. When millions of Tor Browser users visit a site, they all look the same. Install one extension and your fingerprint becomes unique — you're no longer in the crowd.

The only extensions included by default are uBlock Origin (pre-configured, do not change its settings) and NoScript (integrated into the Security Level system). Do not add to these.

The same principle applies to:
- Changing browser fonts or themes
- Modifying `about:config` settings that affect fingerprint
- Installing language packs beyond what Tor Browser ships with

## JavaScript and NoScript

When Security Level is set to Safer or Safest, NoScript controls which sites can run JavaScript. With Safer:

- HTTPS sites: JavaScript allowed by default
- HTTP sites: JavaScript blocked

To temporarily enable JavaScript on a specific HTTP site (if necessary):
1. Click the NoScript icon (S) in the toolbar
2. Click the options menu for the domain
- Select "Temporarily Trusted" (expires when you close the tab or use New Identity)

Never click "Trust all domains" — this disables the entire point of NoScript.

## Handling Downloads

Downloads are a significant risk in Tor Browser. Downloaded files can contain content that your operating system opens automatically — PDFs, documents, videos — and some of these may make network connections that bypass Tor, revealing your real IP.

Tor Browser shows a warning for all downloads:

```
"You are about to open: file.pdf
Opening this file may bypass Tor protections.
You should only open files you trust."
```

Heed this warning. For sensitive downloads:

1. Use Tails OS or Whonix where the entire system routes through Tor
2. Open downloaded files only on an air-gapped machine
3. Use a read-only sandbox (like Dangerzone for documents)
4. For PDFs: use a text extractor rather than a PDF viewer

## Tor Browser on Windows vs Linux vs macOS

Tor Project's security is strongest when running on a minimal Linux system:

**Windows**: Microsoft telemetry, Windows Defender integration, and system services create more attack surface and more potential for de-anonymizing metadata.

**macOS**: Better than Windows but still includes Apple telemetry and network agents.

**Linux**: Preferred. Minimal system daemons, no mandatory telemetry, more transparent system behavior.

**Tails OS**: Best option for maximum anonymity — the entire operating system is designed around anonymity, running from RAM with no persistent storage by default, and all connections go through Tor at the system level (not just Tor Browser).

## Common Mistakes That Undermine Tor Browser

**Logging into personal accounts**: If you log into your real Google, Facebook, or email account inside Tor Browser, the site operator knows who you are. Tor hides your IP from the site, but your account credentials defeat that. Never mix identity with anonymity.

**Using Tor Browser at home for sensitive tasks, then regular browsing**: If you use Tor Browser for anonymous research and then immediately search for your own name in another tab, those sessions should not overlap. Use Tor Browser exclusively for sensitive tasks, not mixed with regular browsing.

**Resizing the browser window**: Tor Browser opens at a specific window size. Resizing creates a unique window dimensions fingerprint. Always use Tor Browser at its default window size or fullscreen (Tor 12+ handles this better with letterboxing).

**Allowing location access**: Never grant location permission to any site inside Tor Browser. Even a coarse location request can reveal which country's Tor exit node you're using.

**Using HTTP sites for sensitive work**: Tor protects against your ISP and network observers. An exit relay can see HTTP traffic in plaintext. Always use HTTPS for sensitive sites — check the address bar for the padlock.

## Bridges for Censored Regions

If Tor is blocked in your country, configure bridges in Tor Browser:

Settings → Tor → Bridges → Use a bridge → Request a bridge from torproject.org

Obfs4 bridges obfuscate Tor traffic to look like random HTTPS traffic. Snowflake uses WebRTC to masquerade as video calls.

```bash
# Request bridges via email (works when the website is blocked)
# Send empty email to: bridges@torproject.org
```

## Related Reading

- [Privacy-Focused Web Browser Comparison 2026](/privacy-browser-comparison-2026/)
- [How to Use Tails OS for Maximum Privacy](/how-to-use-tails-os-for-maximum-privacy-complete-setup-guide/)
- [Wireshark Basics for Privacy Network Analysis](/wireshark-privacy-network-analysis-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
