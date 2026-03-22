---
layout: default
title: "Tor Browser Screen Size Fingerprint Protection"
description: "Learn how Tor Browser protects against screen size fingerprinting and how to configure it for maximum privacy. Practical tips for developers and power"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /tor-browser-screen-size-fingerprint-protection/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Screen size fingerprinting represents one of the most subtle yet effective tracking techniques used across the web. Unlike cookies or IP-based tracking, screen fingerprinting operates passively, collecting display dimensions that can uniquely identify users across sessions without leaving any traceable artifacts. Tor Browser implements several defensive mechanisms to combat this vector, and understanding these protections helps you maintain better anonymity.

## Table of Contents

- [How Screen Size Fingerprinting Works](#how-screen-size-fingerprinting-works)
- [Tor Browser's Defense Mechanisms](#tor-browsers-defense-mechanisms)
- [Configuring Tor Browser for Enhanced Protection](#configuring-tor-browser-for-enhanced-protection)
- [Detecting Fingerprinting Attempts](#detecting-fingerprinting-attempts)
- [Common Pitfalls](#common-pitfalls)
- [Practical Recommendations](#practical-recommendations)
- [Advanced Fingerprinting Vectors](#advanced-fingerprinting-vectors)
- [Testing Your Fingerprint Against Real Datasets](#testing-your-fingerprint-against-real-datasets)
- [Fingerprinting Timing and Behavior](#fingerprinting-timing-and-behavior)
- [Combining Protection Methods for Defense-in-Depth](#combining-protection-methods-for-defense-in-depth)
- [Fingerprinting Across Tor Circuits](#fingerprinting-across-tor-circuits)
- [Limitations and Tradeoffs](#limitations-and-tradeoffs)

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

## Advanced Fingerprinting Vectors

Beyond screen size, multiple fingerprinting techniques attempt to identify users. Tor Browser's defenses extend across multiple vectors.

### Device Pixel Ratio Fingerprinting

The device pixel ratio (physical pixels / CSS pixels) can uniquely identify devices:

```javascript
// What websites collect
console.log(window.devicePixelRatio);
// Typical values: 1.0 (desktop), 2.0 (modern mobile), 3.0 (high-end phones)

// In Tor Browser, this is normalized
// All users report 1.0 regardless of actual hardware
```

Tor Browser normalizes this to 1.0, removing this fingerprinting vector.

### Color Depth Fingerprinting

Monitor color depth varies across devices:

```javascript
// Color depth varies significantly
window.screen.colorDepth        // Typical: 24 (most modern displays)
window.screen.pixelDepth        // Often matches colorDepth

// In Tor Browser, normalized to standard value
// Prevents using color depth as differentiator
```

Tor standardizes color depth to eliminate this distinction.

### Available Screen Dimensions

Websites can measure available screen space (accounting for taskbars):

```javascript
// Available space after OS UI elements
window.screen.availWidth        // Desktop width minus taskbar
window.screen.availHeight       // Height minus menu bar

// In Tor Browser:
// These are masked/normalized to prevent uniqueness
```

Tor Browser's letterboxing ensures consistent available dimensions.

## Testing Your Fingerprint Against Real Datasets

Rather than theoretical fingerprint uniqueness, test against actual datasets of Tor Browser users:

### Using Cover Your Tracks

The EFF's Cover Your Tracks tool (formerly Panopticlick) tests your browser fingerprint against known Tor users:

```
Visit: https://coveryourtracks.eff.org

The tool reports:
- Your unique fingerprint
- Uniqueness against all browsers
- Uniqueness against Tor users specifically
- Which fingerprinting vectors identify you
```

**Interpreting results:**
- "Unique" = Your fingerprint is one-of-a-kind
- "Unique among Tor users" = No Tor user shares your fingerprint
- "Unknown" = Fingerprint data insufficient

For Tor Browser, most users should see low uniqueness scores. If high, check that Tor security features are enabled.

### BrowserLeaks Fingerprinting Test

BrowserLeaks provides detailed fingerprinting analysis:

```
Visit: https://browserleaks.com/

Provides analysis of:
- Canvas fingerprinting resistance
- WebGL capabilities
- WebRTC leak detection
- TLS version and ciphers
- HTTP/2 and ALPN support
- DNS/IP leaks
```

### Running Local Fingerprinting Tests

For developers, fingerprint your Tor Browser programmatically:

```javascript
// Thorough fingerprinting test
const getFingerprint = () => {
    const fingerprint = {
        // Screen dimensions
        screenSize: `${window.screen.width}x${window.screen.height}`,
        innerSize: `${window.innerWidth}x${window.innerHeight}`,

        // Display properties
        devicePixelRatio: window.devicePixelRatio,
        colorDepth: window.screen.colorDepth,

        // Browser properties
        userAgent: navigator.userAgent,
        hardwareConcurrency: navigator.hardwareConcurrency,
        deviceMemory: navigator.deviceMemory,

        // Canvas fingerprint
        canvasFingerprint: getCanvasFingerprint(),

        // Timezone
        timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,

        // Language
        language: navigator.language
    };

    return fingerprint;
};

function getCanvasFingerprint() {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    ctx.textBaseline = 'top';
    ctx.font = '14px Arial';
    ctx.textBaseline = 'alphabetic';
    ctx.fillStyle = '#f60';
    ctx.fillRect(125, 1, 62, 20);
    ctx.fillStyle = '#069';
    ctx.fillText('Tor Browser Fingerprint Test', 2, 15);
    ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
    ctx.fillText('Tor Browser Fingerprint Test', 4, 17);

    return canvas.toDataURL();
}
```

Compare your Tor Browser results against Firefox on the same system. Tor Browser should return significantly more normalized values.

## Fingerprinting Timing and Behavior

Beyond static properties, websites track behavioral patterns:

### Click Pattern Fingerprinting

Websites can create a fingerprint based on how you click, type, and move your mouse:

```javascript
// Behavioral fingerprinting (what bad sites do)
window.addEventListener('mousemove', (e) => {
    // Track mouse movement patterns, speed, acceleration
});

window.addEventListener('keypress', (e) => {
    // Track typing speed and patterns
});

// Tor Browser mitigations:
// - Pointer events disabled in some contexts
// - Typing patterns obscured through event batching
// - Mouse movement tracking limited
```

Tor Browser doesn't completely prevent behavioral tracking, but it introduces enough randomization to make patterns unreliable.

### Timing-Based Fingerprinting

Websites can use performance timing to distinguish browsers:

```javascript
// Performance timing revealing (limited in Tor Browser)
console.time('operation');
// ... do something
console.timeEnd('operation');

// High-resolution timing disabled in Tor Browser
// performance.now() returns less granular values
// Prevents microsecond-level measurements
```

Tor Browser limits timer precision to reduce fingerprinting surface.

## Combining Protection Methods for Defense-in-Depth

No single protection is perfect. Combine multiple techniques:

**Layer 1: Tor Browser defaults**
- Window resizing to standard width
- Letterboxing
- Normalized device properties

**Layer 2: Security level adjustment**
- Safer or Safest mode disables additional features
- Blocks JavaScript, WebGL, others reducing fingerprint surface

**Layer 3: Extensions**
- NoScript provides script blocking
- uBlock Origin blocks fingerprinting scripts
- Canvas fingerprint protection blocks canvas access

**Layer 4: Browser configuration**
- Disable WebGL: `webgl.disabled = true`
- Disable WebRTC: `media.peerconnection.enabled = false`
- Disable hardware acceleration: `layers.acceleration.disabled = true`

Stacking these layers makes fingerprinting exponentially harder.

## Fingerprinting Across Tor Circuits

When you request a new Tor circuit:

```
Old circuit: IP 1.2.3.4, Browser fingerprint A
Renew circuit (Ctrl+Shift+L)
New circuit: IP 5.6.7.8, Browser fingerprint A
```

Your fingerprint stays the same while IP changes. Websites can correlate through fingerprint even after IP rotation.

Advanced fingerprinting systems track:

1. Initial fingerprint
2. Renew Tor circuit (IP changes)
3. Same fingerprint = same user despite different IP

Mitigate by:
- Restart Tor Browser between sessions (resets fingerprint)
- Use separate Tor Browser instances for different identities
- Maximize time between circuit renewals

## Limitations and Tradeoffs

Tor Browser's fingerprinting protection has limits:

**Canvas blocking affects some sites:**
- Signature functionality on PDFs
- Advanced graphics features
- WebGL-dependent visualizations

**Normalized screen size breaks layouts on some sites:**
- Responsive design assumes variable widths
- Some sites detect Tor and restrict access
- Letterboxing appears unusual

**Security level restrictions impact usability:**
- Safest mode disables JavaScript entirely
- Many modern sites require JavaScript
- Productivity applications become unusable

Choose your protection level based on:
- Threat model (are you evading determined adversary?)
- Usability requirements (how much functionality do you need?)
- Time investment (fingerprinting resistance requires active management)

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Tor Browser Fingerprinting Protection How It Makes Everyone](/privacy-tools-guide/tor-browser-fingerprinting-protection-how-it-makes-everyone-/)
- [Tor Browser Canvas Fingerprinting Protection](/privacy-tools-guide/tor-browser-canvas-fingerprinting-protection/)
- [Tor Browser Font Fingerprinting Protection](/privacy-tools-guide/tor-browser-font-fingerprinting-protection/)
- [Browser Fingerprinting Protection Techniques](/privacy-tools-guide/browser-fingerprint-protection-guide)
- [Screen Resolution Fingerprinting Why Changing Display](/privacy-tools-guide/screen-resolution-fingerprinting-why-changing-display-settin/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
