---
layout: default
title: "Tor Browser Security Settings Configuration Guide"
description: "How to configure Tor Browser's security levels, circuit controls, JavaScript settings, and anti-fingerprinting features for different threat scenarios"
date: 2026-03-21
author: theluckystrike
permalink: /tor-browser-security-settings-guide/
categories: [guides]
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

## Advanced Configuration: The Torrc File

For developers and power users, Tor Browser allows direct configuration through `torrc` files. Access the Tor Browser data directory and edit configuration directly:

**macOS/Linux:**
```bash
# Locate Tor Browser profile
open ~/.var/app/torbrowser/data/Tor/

# Or if running Tor separately:
# Edit /etc/tor/torrc
```

**Windows:**
```
C:\Users\[Username]\AppData\Roaming\Tor Browser\Browser\TorBrowser\Data\Tor\torrc
```

**Key configurations for privacy:**

```bash
# Use specific exit nodes only (advanced - can reduce anonymity)
ExitNodes {us},{gb},{ca}  # Only exit through these countries
StrictNodes 1              # Fail if specified exits unavailable

# Disable directory caching to prevent metadata collection
DirCache 0

# Increase circuit timeout for heavy users
CircuitBuildTimeout 120

# Use obfuscated bridges if Tor is blocked
UseBridges 1
ClientTransportPlugin obfs4 exec /path/to/obfs4proxy

# Set SOCKS5 proxy to non-standard port if needed
SocksPort 9999

# Request specific guard nodes (advanced)
UseEntryGuards 1
NumEntryGuards 3
```

## Testing Tor Browser Configuration

After configuration changes, verify your anonymity:

**DNS Leak Test:**
```bash
# Within Tor Browser, visit:
dnsleaktest.com

# All DNS queries should resolve through Tor exit nodes
# NOT through your ISP
```

**IP Leak Test:**
```bash
# Within Tor Browser, visit:
ipleak.net

# Your IP should be the Tor exit node IP, NOT your real IP
# IPv6 addresses should be cleared or only show exit node addresses
```

**WebRTC Leak Test:**
```bash
# Within Tor Browser, visit:
browserleaks.com/webrtc

# Should show only Tor exit node IP
# If shows your ISP IP, WebRTC is leaking
```

## Tor Browser Fingerprinting Risks

Tor Browser specifically minimizes browser fingerprinting, but users often accidentally re-enable it:

**Common fingerprinting mistakes:**

1. **Installing fonts** - Browser fonts are fingerprint vectors. Tor Browser intentionally ships with limited fonts.

2. **Changing default zoom level** - Tor Browser zooms at 100%. Changing zoom creates a unique fingerprint:
   ```
   Ctrl+0 (Windows/Linux) or Cmd+0 (macOS) to reset zoom to default
   ```

3. **Resizing window** - Window size is a fingerprint vector. Use Ctrl+Alt+U for fullscreen with maximum consistency.

4. **Accessing about:config** - Modifying `browser.sessionstore.privacy_level` or other settings creates fingerprints. Don't customize unless absolutely necessary.

5. **Using multiple profiles** - Each profile is a separate fingerprint. Tor Browser intentionally uses one profile per instance.

## Timing Attack Mitigation

Tor protects against IP leaks but not against traffic analysis by powerful network observers:

**Timing attack example:**
```
Observer notes:
- Your home IP connects to Tor at 14:23:15
- Tor exit node connects to target site at 14:23:18 (3 second delay)
- Message sent at 14:23:20
- Your home IP disconnects at 14:24:15
- Tor exit disconnects at 14:24:18

Observer correlates: Your timing matches the exit node timing.
```

**Mitigation:**
- Use Tor Browser during high-traffic periods (avoiding unusual times)
- Avoid accessing Tor right before/after accessing target in clearnet
- Use Tor exclusively for sensitive sessions (don't mix with regular browsing)
- Add random delays to your actions using `xdotool` on Linux or Automator on macOS

**Timing noise script (Linux):**
```bash
#!/bin/bash
# Add random delays to confuse timing attacks

delay=$((RANDOM % 60 + 30))  # Random 30-90 second delay
echo "Waiting ${delay} seconds before action..."
sleep $delay

# Now perform your Tor activity
torsocks curl https://example.com
```

## Using Tails with Tor Browser

Tails OS (The Amnesic Incognito Live System) combines Tor Browser with system-level protections:

**Tails advantages over Tor Browser alone:**
- Entire OS routes through Tor (not just browser)
- No disk storage (runs from RAM)
- Automatic application whitelisting (prevents accidental non-Tor traffic)
- Built-in VPN kill switch (Tor circuit fails if network changes)
- No persistent browsing history

**Running Tails:**
```bash
# Boot Tails from USB or DVD
# Create persistence volume if you need to save settings:
# Use Tails Installer to create persistent storage

# Everything you do in Tails is routed through Tor
# SSH, DNS queries, all network traffic

# On shutdown, Tails clears all RAM
# No evidence of activity remains
```

## When Tor Browser Is Not Enough

Recognize when additional protections are needed:

**When browser anonymity is insufficient:**
- Accessing hidden services from a hostile network (use Tails instead)
- Communicating with extremely high-risk contacts (add ProtonMail encryption)
- Operating in environment with physical surveillance (use Tails on air-gapped machine)
- Expecting state-level adversary (combine Tor + VPN + Tails)

**Tor Browser plus email setup:**
```bash
# ProtonMail in Tor Browser for additional email privacy
# 1. Install Tor Browser
# 2. Access: https://protonmailrmez3lotccipshtkleegetolb73fuirgj7r4o4vfu7ozyd.onion/
#    (ProtonMail's .onion address - only accessible via Tor)
# 3. Create account with security questions only (no recovery email)
# 4. Enable two-factor authentication
# 5. Use Tor Browser exclusively for that account
```

## Troubleshooting Tor Browser Issues

**Issue: Tor Browser won't connect**
```bash
# Check if Tor service is running
ps aux | grep tor

# Restart Tor:
killall tor
# Then restart Tor Browser

# Check torrc for syntax errors:
tor --validate-config -f /etc/tor/torrc
```

**Issue: Very slow connections**
- Try New Circuit (may help with congested relay)
- Use obfuscated bridges (meek, snowflake) which may have better capacity
- Test at different times (relay capacity varies by hour)

**Issue: Websites blocking Tor traffic**
- Some sites block known Tor exit nodes
- Use obfuscated bridges to disguise Tor traffic
- Contact site operators to request Tor support (many major sites now support it)

## Related Reading

- [Privacy-Focused Web Browser Comparison 2026](/privacy-browser-comparison-2026/)
- [How to Use Tails OS for Maximum Privacy](/how-to-use-tails-os-for-maximum-privacy-complete-setup-guide/)
- [Wireshark Basics for Privacy Network Analysis](/wireshark-privacy-network-analysis-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
