---
layout: default
title: "Battery API Fingerprinting: How Battery Status Tracks."
description: "Discover how websites use the Battery API to fingerprint users, the privacy risks involved, and practical ways to protect yourself from this."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /battery-api-fingerprinting-how-battery-status-tracks-you-exp/
categories: [guides, security]
reviewed: true
score: 8
---

The web platform provides numerous APIs designed to enhance user experience, but some of these APIs can be weaponized for tracking. The Battery Status API, standardized under the W3C, exemplifies this tension between functionality and privacy. Originally created to help web applications adjust their behavior based on battery level—dimming screens on low battery or pausing resource-intensive tasks—this API simultaneously exposes data that can be used to fingerprint users across sessions.

## What Is the Battery Status API?

The Battery Status API, also known as the Battery Manager API, provides JavaScript access to battery information on client devices. Implemented in most modern browsers, it exposes four key properties:

- **charging**: A boolean indicating whether the device is currently charging
- **chargingTime**: The time remaining in seconds until the battery is fully charged
- **dischargingTime**: The time remaining in seconds until the battery is fully depleted
- **level**: A number between 0 and 1 representing the current battery percentage

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

Fingerprinting works by collecting enough unique attributes to identify a user without relying on cookies or login credentials. The Battery API contributes several high-entropy signals to this process:

**Precise Battery Level**: When combined with charging status and timestamps, the exact battery percentage creates a short-term identifier. A user at 47% battery while charging differs significantly from one at 47% while discharging. Tracking scripts can poll this value repeatedly, creating a fingerprint that may persist across browsing sessions.

**Charging Patterns**: The timing and frequency of charging events vary substantially between users. Someone who plugs in at 8 AM daily has a different pattern from one who charges only in the evening. These behavioral signals contribute to user profiles.

**Discharge Rates**: The speed at which a battery drains depends on screen brightness, active applications, and hardware characteristics. This rate varies between devices and users, adding another dimension to the fingerprint.

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

**Firefox**: Completely removed the Battery API in 2016 after researchers demonstrated its fingerprinting potential. The API remains unavailable in Firefox desktop and mobile versions.

**Safari**: Limited the API's precision, returning rounded values rather than exact percentages. This reduces entropy while maintaining basic functionality for legitimate use cases.

**Chrome/Chromium**: Implemented the API fully but introduced restrictions. The API only works over HTTPS, and some properties return infinity when the battery state cannot be determined. Recent versions have shown signs of further limitations.

**Edge**: Follows Chrome's implementation since it shares the Chromium engine.

These differences themselves create tracking opportunities. The presence or absence of the Battery API, combined with the precision of returned values, forms another component of the overall fingerprint.

## Privacy Implications

The implications of battery fingerprinting extend beyond simple tracking:

**Cross-Site Tracking**: Unlike cookies, battery-based fingerprints persist across different websites. A user visiting multiple sites can be recognized without any persistent storage on their device.

**Device Identification**: The combination of battery characteristics with other hardware information can uniquely identify specific devices. This affects users who believe they're anonymous while browsing.

**Behavioral Profiling**: Charging patterns and battery drain rates reveal usage habits. Advertisers can infer when users are likely to be near power outlets, traveling, or using specific applications.

**Exploitation of Vulnerable Users**: Users with unusual battery behavior—perhaps due to faulty batteries or unusual charging habits—become particularly easy to identify and track.

## Protecting Against Battery API Tracking

Several strategies help mitigate Battery API fingerprinting:

**Use Privacy-Focused Browsers**: Firefox, Brave, and Tor Browser block or significantly restrict the Battery API. These browsers prioritize user privacy over the API's convenience.

**Disable JavaScript**: While extreme, disabling JavaScript entirely eliminates Battery API access. This approach breaks many websites but provides complete protection.

**Use Browser Extensions**: Privacy-focused extensions like Privacy Badger or uBlock Origin can block known fingerprinting scripts. These tools maintain blocklists updated based on discovered trackers.

**Browser Fingerprinting Protection**: Tools like CanvasBlocker or specialized browser configurations can add noise to browser APIs, making accurate fingerprinting more difficult.

**Regularly Clear Site Data**: While not directly effective against Battery API tracking (since no storage is required), maintaining good cookie hygiene helps reduce the overall tracking footprint.

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

## Conclusion

The Battery API demonstrates how seemingly useful web platform features can become privacy concerns. While the API serves legitimate purposes, its fingerprinting potential is well-documented. Users concerned about tracking should consider browsers that restrict or block this API, while developers should evaluate whether their use cases genuinely require battery information. As web privacy continues to evolve, the tension between functionality and anonymity remains a central challenge.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
