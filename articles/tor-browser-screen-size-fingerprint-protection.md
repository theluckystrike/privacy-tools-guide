---

layout: default
title: "Tor Browser Screen Size Fingerprint Protection"
description: "Learn how Tor Browser protects against screen size fingerprinting and how to configure it for maximum privacy. Practical tips for developers and power."
date: 2026-03-15
author: theluckystrike
permalink: /tor-browser-screen-size-fingerprint-protection/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Screen size fingerprinting represents one of the most subtle yet effective tracking techniques used across the web. Unlike cookies or IP-based tracking, screen fingerprinting operates passively, collecting display dimensions that can uniquely identify users across sessions without leaving any traceable artifacts. Tor Browser implements several defensive mechanisms to combat this vector, and understanding these protections helps you maintain better anonymity.

## How Screen Size Fingerprinting Works

When you visit a website, your browser exposes various display properties through the JavaScript Window and Screen APIs. The most commonly collected values include:

- `window.screen.width` and `window.screen.height` — your total screen resolution
- `window.innerWidth` and `window.innerHeight` — the viewport size within browser chrome
- `window.outerWidth` and `window.outerHeight` — the entire browser window dimensions
- `window.devicePixelRatio` — the ratio between physical and CSS pixels

These values, when combined, create a surprisingly unique identifier. A user with a 1920×1080 display running a maximized browser window at 1914×1043 pixels generates a distinct fingerprint. Tracking companies correlate these values across sessions, building persistent profiles even when users clear cookies or use VPN services.

The technique works because most users have common but not identical configurations. Browser toolbars, operating system taskbars, and window management behaviors create variance that narrows down the possible user pool dramatically.

## Tor Browser's Defense Mechanisms

Tor Browser addresses screen size fingerprinting through two primary strategies: window resizing and letterboxing.

### Window Resizing

By default, Tor Browser constrains the content area to a standard width while allowing the actual window to vary. The browser reports `window.innerWidth` as 1000 pixels regardless of your actual viewport size, creating a consistent baseline across Tor users. This approach prevents websites from distinguishing users based on viewport dimensions.

To check this behavior yourself, open Tor Browser and execute the following in the developer console:

```javascript
console.log('innerWidth:', window.innerWidth);
console.log('innerHeight:', window.innerHeight);
console.log('outerWidth:', window.outerWidth);
console.log('outerHeight:', window.outerHeight);
```

You will notice that `innerWidth` remains constant at 1000 pixels, while the outer dimensions reflect your actual window size.

### Letterboxing

Letterboxing adds gray margins around web content when the window size exceeds the content area. This technique serves two purposes: it maintains the reported 1000-pixel width while preserving the ability to view content at larger resolutions, and it prevents websites from detecting the actual available viewport by measuring content overflow behaviors.

The letterboxing implementation in Tor Browser adds equal margins on all sides when necessary, ensuring the content area remains centered and the reported dimensions stay consistent.

## Configuring Tor Browser for Enhanced Protection

While Tor Browser's defaults provide reasonable protection, power users can adjust settings for specific threat models.

### Adjusting Security Levels

Tor Browser includes a security slider with three levels:

**Standard** — All Tor Browser features enabled, providing the best browsing experience while maintaining privacy protections.

**Safer** — Disables JavaScript on non-HTTPS sites, removes some font and rendering features, and sets stricter content security policies. This level significantly reduces the fingerprinting surface but may break some websites.

**Safest** — Disables all JavaScript entirely, providing maximum protection against fingerprinting but severely limiting website functionality.

Access this slider through the shield icon in the address bar or through `about:preferences#privacy`.

### Manual Window Size Control

For specialized use cases, you can force Tor Browser to use specific window dimensions by modifying `about:config`:

1. Type `about:config` in the address bar
2. Search for `privacy.resistFingerprinting`
3. Set this to `true` if not already enabled
4. Create a new integer value named `privacy.resistFingerprinting.window.width`
5. Set your desired width (Tor Browser will round to nearest 100 pixels)

This approach trades some usability for consistency. Websites will see exactly the dimensions you specify, making your fingerprint more stable across sessions.

## Detecting Fingerprinting Attempts

Developers building privacy-aware applications may want to detect when fingerprinting occurs. Tor Browser includes protections against some detection methods, but understanding the techniques helps you recognize suspicious behavior.

A basic detection script might look like:

```javascript
function detectScreenFingerprint() {
  const screenData = {
    width: window.screen.width,
    height: window.screen.height,
    availWidth: window.screen.availWidth,
    availHeight: window.screen.availHeight,
    colorDepth: window.screen.colorDepth,
    pixelDepth: window.screen.pixelDepth,
    innerWidth: window.innerWidth,
    innerHeight: window.innerHeight,
    outerWidth: window.outerWidth,
    outerHeight: window.outerHeight,
    devicePixelRatio: window.devicePixelRatio
  };
  
  return screenData;
}
```

In standard browsers, this returns your actual values. In Tor Browser with fingerprinting resistance enabled, many values are normalized or obscured.

## Common Pitfalls

Users often inadvertently reduce their anonymity through well-intentioned but counterproductive behaviors.

**Maximizing windows** — While intuitive for productivity, maximized windows create unique fingerprints. The combination of your screen resolution plus maximized dimensions narrows your anonymity set significantly. Tor Browser's resizing helps, but avoiding full maximization maintains consistency.

**Resizing continuously** — Rapid window resizing generates a timeline of dimension changes that can serve as a behavioral fingerprint. Websites can track this pattern across sessions.

**Using multiple windows** — Opening multiple Tor Browser windows at different sizes creates conflicting fingerprints. Each window may report slightly different values, making correlation easier.

**Disabling resistance features** — Some users disable `privacy.resistFingerprinting` to fix website layout issues, inadvertently exposing their actual screen characteristics.

## Practical Recommendations

For developers and power users seeking optimal protection:

1. **Accept default settings** — Tor Browser's defaults balance usability with privacy effectively for most users.

2. **Use the Safer security level** — This provides substantial fingerprinting protection without completely breaking JavaScript-dependent sites.

3. **Avoid window manipulation** — Keep your Tor Browser window at a consistent size rather than resizing frequently.

4. **Test your fingerprint** — Use tools like Panopticlick or Cover Your Tracks to verify your protection level, though recognize these tests have limitations.

5. **Understand the tradeoffs** — Maximum protection sometimes means reduced functionality. Accept this compromise based on your specific threat model.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Tor Browser Canvas Fingerprinting Protection](/privacy-tools-guide/tor-browser-canvas-fingerprinting-protection/)
- [Tor Browser Fingerprinting Protection: How It Makes.](/privacy-tools-guide/tor-browser-fingerprinting-protection-how-it-makes-everyone-/)
- [How to Check Your Browser Fingerprint Uniqueness Score.](/privacy-tools-guide/how-to-check-your-browser-fingerprint-uniqueness-score-onlin/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
