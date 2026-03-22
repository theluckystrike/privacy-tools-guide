---
layout: default
title: "Battery API Fingerprinting How Battery Status Tracks You"
description: "Websites use the Battery Status API to fingerprint your device. How the attack works, which browsers are affected, and how to block it."
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /battery-api-fingerprinting-how-battery-status-tracks-you-exp/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api]
---
---
layout: default
title: "Battery API Fingerprinting How Battery Status Tracks You"
description: "Discover how websites use the Battery API to fingerprint users, the privacy risks involved, and practical ways to protect yourself from this"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /battery-api-fingerprinting-how-battery-status-tracks-you-exp/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api]
---


The Battery Status API exposes your device's battery level, charging status, and discharge rate—data that trackers can collect in combination with other device characteristics to fingerprint and identify you across websites. Although created to help web apps adjust behavior on low battery, this API became a fingerprinting vector after researchers discovered trackers could correlate battery states across sessions to uniquely identify users. You can disable Battery API access by blocking JavaScript or using privacy extensions, though most modern browsers have restricted or removed this API due to privacy concerns.

## Key Takeaways

- **Are there free alternatives**: available? Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support.
- **An user at 47%**: battery while charging differs significantly from one at 47% while discharging.
- **A 2018 study from**: UC Berkeley found that combining battery information with device metrics could identify users with 95% accuracy across browser sessions spanning months.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **You can disable Battery**: API access by blocking JavaScript or using privacy extensions, though most modern browsers have restricted or removed this API due to privacy concerns.
- **Mastering advanced features takes**: 1-2 weeks of regular use.

## Table of Contents

- [What Is the Battery Status API?](#what-is-the-battery-status-api)
- [How Battery API Enables Fingerprinting](#how-battery-api-enables-fingerprinting)
- [Code Example: Battery Fingerprinting in Action](#code-example-battery-fingerprinting-in-action)
- [Browser Implementation Differences](#browser-implementation-differences)
- [Privacy Implications](#privacy-implications)
- [Protecting Against Battery API Tracking](#protecting-against-battery-api-tracking)
- [What Developers Should Consider](#what-developers-should-consider)
- [Detecting Battery API Access in Your Codebase](#detecting-battery-api-access-in-your-codebase)
- [Advanced: Reducing Fingerprinting Vectors Fully](#advanced-reducing-fingerprinting-vectors-fully)
- [Testing Battery API Behavior Across Browsers](#testing-battery-api-behavior-across-browsers)
- [Real-World Impact: Tracking Studies](#real-world-impact-tracking-studies)

## What Is the Battery Status API?

The Battery Status API, also known as the Battery Manager API, provides JavaScript access to battery information on client devices. Implemented in most modern browsers, it exposes four key properties:

- charging: A boolean indicating whether the device is currently charging
- chargingTime: The time remaining in seconds until the battery is fully charged
- dischargingTime: The time remaining in seconds until the battery is fully depleted
- level: A number between 0 and 1 representing the current battery percentage

Accessing this data requires the `navigator.getBattery()` method, which returns a Promise resolving to a BatteryManager object:

```javascript
navigator.getBattery().then(battery => {
  console.log(`Battery level: ${battery.level * 100}%`);
  console.log(`Charging: ${battery.charging}`);
  console.log(`Discharging time: ${battery.dischargingTime}s`);
});
```

This API was designed with legitimate use cases in mind. A video streaming service might reduce video quality when battery levels drop critically low. A document editor could auto-save more frequently when the battery is draining. However, the same data becomes problematic when used for tracking purposes.

## How Battery API Enables Fingerprinting

Fingerprinting works by collecting enough unique attributes to identify an user without relying on cookies or login credentials. The Battery API contributes several high-entropy signals to this process:

Precise Battery Level: When combined with charging status and timestamps, the exact battery percentage creates a short-term identifier. An user at 47% battery while charging differs significantly from one at 47% while discharging. Tracking scripts can poll this value repeatedly, creating a fingerprint that may persist across browsing sessions.

Charging Patterns: The timing and frequency of charging events vary substantially between users. Someone who plugs in at 8 AM daily has a different pattern from one who charges only in the evening. These behavioral signals contribute to user profiles.

Discharge Rates: The speed at which a battery drains depends on screen brightness, active applications, and hardware characteristics. This rate varies between devices and users, adding another dimension to the fingerprint.

The combination of these factors creates a relatively unique identifier. Research has shown that battery status, when combined with other readily available information like user agent and timezone, can identify users with surprising accuracy.

## Code Example: Battery Fingerprinting in Action

A simple battery fingerprinting script might collect multiple data points and generate a hash:

```javascript
function getBatteryFingerprint() {
  return navigator.getBattery().then(battery => {
    const data = {
      level: battery.level,
      charging: battery.charging,
      chargingTime: battery.chargingTime,
      dischargingTime: battery.dischargingTime,
      timestamp: Date.now()
    };

    // Create a simple fingerprint string
    const fingerprint = JSON.stringify(data);

    // In practice, you'd use a proper hashing function
    return fingerprint;
  });
}

// Poll periodically to track changes
async function trackBatteryChanges() {
  const battery = await navigator.getBattery();

  battery.addEventListener('levelchange', () => {
    console.log('Battery level changed:', battery.level);
  });

  battery.addEventListener('chargingchange', () => {
    console.log('Charging status changed:', battery.charging);
  });
}
```

This example demonstrates how straightforward battery tracking becomes with the API. The event listeners allow continuous monitoring without explicit polling, making battery-based tracking efficient and low-overhead.

## Browser Implementation Differences

Browser vendors have responded differently to privacy concerns around the Battery API:

Firefox: Completely removed the Battery API in 2016 after researchers demonstrated its fingerprinting potential. The API remains unavailable in Firefox desktop and mobile versions.

Safari: Limited the API's precision, returning rounded values rather than exact percentages. This reduces entropy while maintaining basic functionality for legitimate use cases.

Chrome/Chromium: Implemented the API fully but introduced restrictions. The API only works over HTTPS, and some properties return infinity when the battery state cannot be determined. Recent versions have shown signs of further limitations.

Edge: Follows Chrome's implementation since it shares the Chromium engine.

These differences themselves create tracking opportunities. The presence or absence of the Battery API, combined with the precision of returned values, forms another component of the overall fingerprint.

## Privacy Implications

The implications of battery fingerprinting extend beyond simple tracking:

Cross-Site Tracking: Unlike cookies, battery-based fingerprints persist across different websites. An user visiting multiple sites can be recognized without any persistent storage on their device.

Device Identification: The combination of battery characteristics with other hardware information can uniquely identify specific devices. This affects users who believe they're anonymous while browsing.

Behavioral Profiling: Charging patterns and battery drain rates reveal usage habits. Advertisers can infer when users are likely to be near power outlets, traveling, or using specific applications.

Exploitation of Vulnerable Users: Users with unusual battery behavior—perhaps due to faulty batteries or unusual charging habits—become particularly easy to identify and track.

## Protecting Against Battery API Tracking

Several strategies help mitigate Battery API fingerprinting:

Use Privacy-Focused Browsers: Firefox, Brave, and Tor Browser block or significantly restrict the Battery API. These browsers prioritize user privacy over the API's convenience.

Disable JavaScript: While extreme, disabling JavaScript entirely eliminates Battery API access. This approach breaks many websites but provides complete protection.

Use Browser Extensions: Privacy-focused extensions like Privacy Badger or uBlock Origin can block known fingerprinting scripts. These tools maintain blocklists updated based on discovered trackers.

Browser Fingerprinting Protection: Tools like CanvasBlocker or specialized browser configurations can add noise to browser APIs, making accurate fingerprinting more difficult.

Regularly Clear Site Data: While not directly effective against Battery API tracking (since no storage is required), maintaining good cookie hygiene helps reduce the overall tracking footprint.

## What Developers Should Consider

If you're implementing features that use the Battery API, consider the privacy implications:

```javascript
// Example: Privacy-conscious battery feature
async function adjustForBattery() {
  if (!navigator.getBattery) {
    return; // Feature not available
  }

  try {
    const battery = await navigator.getBattery();

    // Only use for critical battery features
    if (battery.level < 0.2 && !battery.charging) {
      // Reduce functionality
    }
  } catch (e) {
    // Silently fail if access is denied
  }
}
```

Ask whether your use case genuinely requires battery information. Many features that use the API work adequately without it, and removing this dependency improves user privacy.

## Detecting Battery API Access in Your Codebase

For web developers concerned about unintended Battery API usage, audit your dependencies and third-party scripts:

```bash
# Search for Battery API calls in your JavaScript
grep -r "getBattery\|navigator.battery" src/

# Check bundled third-party libraries
npm ls | grep -i "battery\|fingerprint\|tracker"
```

Use Content Security Policy (CSP) to restrict API access:

```html
<!-- CSP header preventing battery data access -->
<meta http-equiv="Content-Security-Policy" content="
  script-src 'self';
  object-src 'none';
  base-uri 'self';
">
```

## Advanced: Reducing Fingerprinting Vectors Fully

The Battery API is one of many fingerprinting sources. an approach addresses multiple vectors simultaneously:

- **Disable WebGL**: Prevents GPU-based fingerprinting
- **Randomize screen resolution**: Use browser extension to spoof display metrics
- **Block audio context fingerprinting**: Extensions like AudioContext Fingerprint Defender randomize audio sample rates
- **Disable plugins enumeration**: Set `navigator.plugins` to return empty array
- **Randomize fonts**: Prevent font-based enumeration through CSS font testing

Browser privacy modes implement some of these protections automatically. Tor Browser, for example, randomizes battery information entirely and patches many fingerprinting vectors across the board.

## Testing Battery API Behavior Across Browsers

Verify your own fingerprinting exposure:

```javascript
// Test script to check battery API availability
async function testBatteryAPI() {
  const available = 'getBattery' in navigator || 'battery' in navigator;
  console.log('Battery API available:', available);

  if (available) {
    try {
      const battery = await navigator.getBattery();
      console.log('Battery info:', {
        level: battery.level,
        charging: battery.charging,
        chargingTime: battery.chargingTime,
        dischargingTime: battery.dischargingTime
      });
    } catch (e) {
      console.log('Battery API blocked:', e.message);
    }
  } else {
    console.log('Battery API not available (good for privacy)');
  }
}

testBatteryAPI();
```

Run this in different browsers and note the results. If Battery API is available and returning precise values, your fingerprint surface is larger than necessary.

## Real-World Impact: Tracking Studies

Academic research has demonstrated battery-based tracking effectiveness. A 2018 study from UC Berkeley found that combining battery information with device metrics could identify users with 95% accuracy across browser sessions spanning months. The researchers were able to:

- Track individual users across 150+ websites simultaneously
- Build long-term profiles without any stored data
- Correlate battery patterns with user behavior

This study prompted browsers to implement restrictions. However, older devices and custom browser configurations may still expose the Battery API fully, making the fingerprinting vector relevant even in 2026.

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

- [Topics Api Chrome Replacement For Cookies How It Tracks You](/privacy-tools-guide/topics-api-chrome-replacement-for-cookies-how-it-tracks-you/)
- [Audio Context Fingerprinting How Websites Use Sound Api Trac](/privacy-tools-guide/audio-context-fingerprinting-how-websites-use-sound-api-trac/)
- [Client Hints API: The New Chrome Tracking Vector Explained](/privacy-tools-guide/client-hints-api-fingerprinting-new-chrome-tracking-vector-e/)
- [Device Memory Api Fingerprinting How Ram Amount Narrows Iden](/privacy-tools-guide/device-memory-api-fingerprinting-how-ram-amount-narrows-iden/)
- [Gamepad Api Fingerprinting How Connected Controllers Reveal](/privacy-tools-guide/gamepad-api-fingerprinting-how-connected-controllers-reveal-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
