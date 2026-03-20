---

layout: default
title: "Firefox Privacy Settings Guide 2026: A Practical Guide."
description: "Master Firefox privacy settings in 2026. This guide covers about:config tweaks, resistFingerprinting, container extensions, and advanced configurations."
date: 2026-03-15
author: theluckystrike
permalink: /firefox-privacy-settings-guide-2026/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Start by opening `about:config` and setting `privacy.resistFingerprinting` to `true`, `network.cookie.cookieBehavior` to `1` (block third-party cookies), and `network.trr.mode` to `2` (enable DNS-over-HTTPS with fallback). These three changes form the core of Firefox privacy hardening in 2026. This guide covers every essential `about:config` tweak, extension recommendation, and profile management strategy for developers and power users.

## Accessing Advanced Settings

The about:config interface exposes Firefox's internal configuration. Type `about:config` in your address bar and press Enter. You'll see a warning—accept it to proceed. The search bar at the top makes finding specific preferences straightforward.

## Core Privacy Preferences

Several settings form the foundation of Firefox's privacy posture. Start with these critical preferences:

**privacy.resistFingerprinting** (Boolean, default: false)

Enabling this setting normalizes your browser's fingerprint by reporting generic system information rather than unique values. It adjusts viewport size, timezone, and font availability.

```
privacy.resistFingerprinting = true
```

When enabled, Firefox reports a standard viewport and limits JavaScript access to detailed timing information. This protects against fingerprinting attacks that identify users based on hardware and configuration differences.

**privacy.trackingprotection.enabled** (Boolean, default: true)

Built-in tracking protection blocks known trackers. Ensure this remains enabled:

```
privacy.trackingprotection.enabled = true
```

Firefox maintains a list of known trackers and automatically blocks requests to those domains. This reduces data leakage to advertisers and third-party analytics providers.

## Network and Connection Settings

Network-related settings control how Firefox communicates with servers:

**network.cookie.cookieBehavior** (Integer, default: 0)

This setting controls cookie handling:
- 0: Accept all cookies
- 1: Accept only from visited domains (recommended)
- 2: Block all cookies
- 3: Block third-party cookies

For maximum privacy, set:

```
network.cookie.cookieBehavior = 1
```

**network.cookie.thirdparty.sessionOnly** (Boolean, default: false)

Session-only third-party cookies expire when you close the browser:

```
network.cookie.thirdparty.sessionOnly = true
```

**network.cookie.lifetimePolicy** (Integer, default: 0)

Controls cookie expiration:
- 0: Accept normally
- 1: Prompt for each cookie
- 2: Accept for current session
- 3: Accept for N days

```
network.cookie.lifetimePolicy = 2
```

## WebGL and Canvas Protection

WebGL and HTML Canvas APIs can leak hardware information:

**webgl.disabled** (Boolean, default: false)

Disabling WebGL prevents WebGL-based fingerprinting but may break some web applications:

```
webgl.disabled = true
```

**canvas.privacy.image_cipher** (Boolean, default: true)

This setting adds noise to canvas readouts, making fingerprinting less reliable:

```
canvas.privacy.image_cipher = true
```

## Referrer and Header Control

Control what information Firefox sends to websites:

**network.http.referer.XSSEnabled** (Boolean, default: true)

Controls referrer sending for cross-site requests:

```
network.http.referer.XSSEnabled = false
```

**network.http.referer.spoofSource** (Boolean, default: false)

When true, Firefox spoofs the referrer to match the destination:

```
network.http.referer.spoofSource = true
```

**privacy.reduceTimerPrecision** (Boolean, default: true)

Reduces timing precision to prevent timing-based attacks:

```
privacy.reduceTimerPrecision = true
```

## DNS and HTTPS Settings

Modern DNS and HTTPS configurations improve privacy:

**network.trr.mode** (Integer, default: 0)

DNS-over-HTTPS (DoH) encrypts your DNS queries:
- 0: System default
- 1: DoH disabled
- 2: DoH enabled, fallback to system DNS
- 3: DoH enabled, no fallback
- 4: DoH enabled with race

```
network.trr.mode = 2
```

**network.trr.uri** (String)

Specify your DoH provider. The default is Cloudflare, but you can use other providers:

```
network.trr.uri = https://dns.google/dns-query
```

**security.tls.version.min** (Integer, default: 1)

Enforce minimum TLS version:

```
security.tls.version.min = 3  # TLS 1.2 minimum
```

## Extension Recommendations

Beyond about:config, Firefox's extension ecosystem enhances privacy:

**uBlock Origin** blocks advertisements and trackers at the network level. Install from addons.mozilla.org and enable built-in filter lists.

**Facebook Container** isolates Facebook tracking to prevent cross-site tracking. Firefox includes this natively—enable it in Preferences > Privacy & Security.

**Multi-Account Containers** create isolated sessions for different purposes. Use separate containers for work, personal browsing, and testing:

```bash
# Containers can be managed via Firefox's built-in container tabs
# Right-click a tab > "Move to Container" > select container
```

## Automation and Profiles

Developers often need multiple browser configurations. Firefox profiles solve this:

```bash
# Create a new profile
firefox -P

# Launch with specific profile
firefox -P "privacy-dev"

# Profile manager
firefox --ProfileManager
```

Each profile maintains separate settings, extensions, and cookies. Create dedicated profiles for development, testing, and sensitive browsing.

## Verification and Testing

After configuring settings, verify they work:

1. Visit `about:config` and search for your modified preferences
2. Check that values match your intended configuration
3. Test in a fresh profile to confirm changes propagate correctly
4. Use browser fingerprinting test sites to evaluate your privacy posture

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
