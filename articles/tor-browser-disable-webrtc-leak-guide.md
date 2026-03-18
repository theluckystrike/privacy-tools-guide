---
layout: default
title: "How to Disable WebRTC Leaks in Tor Browser: A Developer's Guide"
description: "Learn how to identify and mitigate WebRTC IP address leaks in Tor Browser. Practical techniques for developers and power users concerned about privacy."
date: 2026-03-15
author: theluckystrike
permalink: /tor-browser-disable-webrtc-leak-guide/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
Disable WebRTC in Tor Browser by navigating to `about:config` and setting `media.peerconnection.enabled` to `false`, or create a `user.js` file in your profile directory to apply the setting automatically on each launch. While Tor Browser's default protections handle casual privacy needs, power users and developers facing targeted threats can add WebRTC blocking to prevent IP address leaks—though this changes your browser fingerprint and breaks features like video calling. This guide explains the threat model and implementation methods.

## Understanding WebRTC Leaks

WebRTC allows browsers to establish direct connections between peers for features like video calling, file sharing, and live streaming. To establish these connections, WebRTC must discover the user's IP addresses—including those not exposed through the standard VPN or Tor circuit.

When WebRTC is enabled, the browser responds to STUN (Session Traversal Utilities for NAT) requests with both the public IP address and potentially local network addresses. This occurs regardless of the Tor network's proxy settings, creating a potential information leak that can de-anonymize users.

The issue is particularly concerning because WebRTC operates at a lower level than typical browser APIs. Even when using Tor Browser's built-in protections, the STUN requests can bypass the standard proxy configuration, exposing real IP addresses to websites that know how to query the WebRTC API.

## Identifying WebRTC Leaks

Before implementing fixes, verify whether your browser is vulnerable. Several online tools can test for WebRTC leaks, but for developers, creating a simple test provides more control over the verification process.

Create an HTML file to test WebRTC leak detection:

```javascript
function findIP() {
  const ips = [];
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });
  
  pc.createDataChannel('');
  pc.createOffer().then(offer => pc.setLocalDescription(offer));
  
  pc.onicecandidate = (ice) => {
    if (!ice || !ice.candidate || !ice.candidate.candidate) {
      console.log('IPs found:', ips);
      return;
    }
    const line = ice.candidate.candidate.split(' ')[4];
    const parts = line.split(':');
    if (parts.length === 2) {
      ips.push(parts[1]);
    }
  };
}
```

This JavaScript snippet attempts to gather IP addresses through WebRTC ICE candidates. If it returns any IP addresses, your browser has a WebRTC leak.

## Methods to Disable WebRTC in Tor Browser

Tor Browser does not provide a simple checkbox to disable WebRTC. The development team made this decision because completely blocking WebRTC would make users more identifiable—the absence of WebRTC support becomes a fingerprinting vector. However, for users who need additional protection, several approaches exist.

### Method 1: about:config Modifications

Navigate to `about:config` in Tor Browser's address bar. Accept the warning about voiding your warranty. Search for `media.peerconnection.enabled` and set it to `false`.

This change disables all WebRTC functionality. However, Tor Browser may reset this setting when you update the browser or clear your identity. Consider creating a user.js file to apply this setting automatically on each launch.

Create a user.js file in your Tor Browser profile directory:

```javascript
// user.js - Tor Browser user preferences
user_pref("media.peerconnection.enabled", false);
user_pref("media.peerconnection.turn.disable", true);
user_pref("media.peerconnection.use_document_iceservers", false);
```

Place this file in the profile directory (typically found in the TorBrowser/Data/Browser/profile.default folder). On each launch, Tor Browser will apply these preferences.

### Method 2: Browser Extension Approach

For users who prefer not to modify about:config, WebRTC-blocking extensions provide an alternative. However, be cautious—extensions themselves can serve as fingerprinting vectors in Tor Browser. Use well-maintained, minimal extensions from trusted sources.

The WebRTC Control extension or similar tools can toggle WebRTC support on and off. Remember that any extension you add becomes part of your browser fingerprint, potentially making you more identifiable.

### Method 3: Content Script Blocking

For developers building privacy-focused applications, blocking WebRTC at the content script level provides another option. Add a content security policy that prevents WebRTC initialization:

```html
<meta http-equiv="Content-Security-Policy" content="media-src 'none'; connect-src 'none';">
```

This CSP header prevents WebRTC from establishing connections entirely. However, this method only works if you control the web application and won't protect against leaks on third-party websites.

## Trade-offs and Considerations

Disabling WebRTC has consequences beyond breaking video calling features. Some websites use WebRTC for legitimate purposes like file transfer, collaborative editing, and real-time updates. After disabling WebRTC, these features will not function.

Additionally, as mentioned earlier, disabling WebRTC changes your browser fingerprint. Tor Browser's anti-fingerprinting measures attempt to make all users appear identical, but adding extensions or modifying settings can make your browser configuration unique. This paradox means that while you're protecting against IP leaks, you might become more identifiable through browser fingerprinting.

For most users, the default Tor Browser settings provide adequate protection against casual WebRTC exploitation. The threat model matters: if you're facing sophisticated adversaries with the ability to inject JavaScript into websites you visit, additional measures become necessary. For casual browsing and standard privacy needs, Tor Browser's built-in protections are sufficient.

## Verifying Your Protection

After implementing any of these methods, verify that WebRTC is properly disabled. Use multiple testing approaches:

First, check that `media.peerconnection.enabled` shows as false in about:config. Second, test with JavaScript-based leak detection tools. Third, check browser console output for any WebRTC-related errors or warnings when visiting websites that use the technology.

Regular verification is important because browser updates can reset some settings. Maintain a checklist of your privacy configurations and review them after each Tor Browser update.

## Conclusion

WebRTC leaks represent a real but often misunderstood threat to Tor Browser users. While the technology provides useful features for web developers, the privacy implications require careful consideration. For developers and power users willing to accept the trade-offs, disabling WebRTC provides additional protection against IP address leakage.

The key is understanding your threat model. Most users do not need to disable WebRTC, but those handling sensitive information or facing targeted attacks should implement these protections. Remember that browser fingerprinting works both ways—any customization makes you more unique, so balance privacy improvements against the cost of increased identifiability.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
