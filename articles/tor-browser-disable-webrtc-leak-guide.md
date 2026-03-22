---
layout: default
title: "How to Disable WebRTC Leaks in Tor Browser"
description: "Learn how to identify and mitigate WebRTC IP address leaks in Tor Browser. Practical techniques for developers and power users concerned about privacy"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /tor-browser-disable-webrtc-leak-guide/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---
---
layout: default
title: "How to Disable WebRTC Leaks in Tor Browser"
description: "Learn how to identify and mitigate WebRTC IP address leaks in Tor Browser. Practical techniques for developers and power users concerned about privacy"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /tor-browser-disable-webrtc-leak-guide/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---

{% raw %}

Disable WebRTC in Tor Browser by navigating to `about:config` and setting `media.peerconnection.enabled` to `false`, or create a `user.js` file in your profile directory to apply the setting automatically on each launch. While Tor Browser's default protections handle casual privacy needs, power users and developers facing targeted threats can add WebRTC blocking to prevent IP address leaks—though this changes your browser fingerprint and breaks features like video calling. This guide explains the threat model and implementation methods.

## Key Takeaways

- **For most users**: the default Tor Browser settings provide adequate protection against casual WebRTC exploitation.
- **The development team made this decision because completely blocking WebRTC would make users more identifiable**: the absence of WebRTC support becomes a fingerprinting vector.
- **Tor Browser's anti-fingerprinting measures**: attempt to make all users appear identical, but adding extensions or modifying settings can make your browser configuration unique.
- **Use netstat to monitor**: for suspicious connections # Monitor for STUN server connections (typically port 3478, 5349) sudo lsof -i | grep -E "3478|5349|stun" # 4.
- **To establish these connections, WebRTC must discover the user's IP addresses**: including those not exposed through the standard VPN or Tor circuit.
- **This occurs regardless of**: the Tor network's proxy settings, creating a potential information leak that can de-anonymize users.

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

## Threat Model Assessment

Understanding when WebRTC disabling is necessary requires honest threat modeling:

| Threat Scenario | Risk Level | WebRTC Disabling | Better Approach |
|-----------------|-----------|-----------------|-----------------|
| Casual browsing, standard privacy | LOW | Not necessary | Default Tor Browser |
| Journalist in hostile regime | HIGH | Yes, essential | Disable + use Bridge |
| Developer testing privacy tools | MEDIUM | Conditional | Test with both states |
| Corporate security researcher | MEDIUM | Yes | Disable + network monitoring |
| Activist in surveillance state | CRITICAL | Yes | Disable + air-gap testing |

If your threat model involves sophisticated state-level adversaries, disabling WebRTC is foundational. For standard privacy-conscious users, the default Tor Browser configuration provides adequate protection.

## Advanced Configuration with Hardened user.js

A privacy-focused user.js extends beyond WebRTC:

```javascript
// user.js - Detailed Tor Browser hardening
// Place in TorBrowser/Data/Browser/profile.default/user.js

// Disable WebRTC entirely
user_pref("media.peerconnection.enabled", false);
user_pref("media.peerconnection.turn.disable", true);
user_pref("media.peerconnection.use_document_iceservers", false);
user_pref("media.peerconnection.identity.enabled", false);

// DNS-over-HTTPS configuration
user_pref("network.trr.mode", 2);  // DoH-only mode
user_pref("network.trr.uri", "https://9.9.9.9/dns-query");

// Disable other potential leaks
user_pref("network.proxy.socks_remote_dns", true);
user_pref("privacy.resistFingerprinting", true);
user_pref("privacy.window.maxInnerWidth", 1600);
user_pref("privacy.window.maxInnerHeight", 900);

// Disable IndexedDB
user_pref("dom.indexedDB.enabled", false);

// Strict tracking protection
user_pref("privacy.trackingprotection.enabled", true);
user_pref("privacy.trackingprotection.pbmode.enabled", true);
```

Place this file in your Tor Browser profile directory and Tor Browser will apply these settings automatically on each startup.

## Verification Commands for Linux and macOS

testing ensures your WebRTC configuration is effective:

```bash
# 1. Check about:config setting (manual verification required)
# Navigate to about:config in Tor Browser and search for "media.peerconnection.enabled"

# 2. Check system logs for WebRTC errors
# macOS
log stream --predicate 'process == "Tor Browser"' | grep -i webrtc

# Linux
journalctl -u tor | grep -i webrtc

# 3. Use netstat to monitor for suspicious connections
# Monitor for STUN server connections (typically port 3478, 5349)
sudo lsof -i | grep -E "3478|5349|stun"

# 4. Test DNS leaks separately
# Install and run dnsleak test
curl -s https://api.ipleak.net/json

# 5. Monitor Firefox network calls
# Use browser developer tools (F12) and check Network tab
# No XHR or WebSocket calls to STUN servers should appear
```

## Platform-Specific Implementation

### Windows Installation (NSIS Setup)

If using Windows, locate the profile folder in a different path:

```
C:\Users\[USERNAME]\AppData\Local\Tor Browser\Browser\profile.default
```

Copy your `user.js` to this directory. Ensure file permissions allow the Tor Browser process to read the file.

### Linux AppImage or Snap

For AppImage distributions:

```bash
# Extract AppImage to access profile directory
./TorBrowser-[version].x86_64.AppImage --appimage-extract

# Profile location in extracted folder
./squashfs-root/opt/tor-browser/Browser/profile.default

# For Snap installations
~/snap/tor-browser/common/.tor-browser-[version]/Browser/profile.default
```

### macOS with Homebrew

If installed via Homebrew:

```bash
# Locate the profile directory
find ~/Library -name "profile.default" -path "*/Tor Browser*"

# Typically located at
~/Library/Application\ Support/Tor\ Browser/Browser/profile.default
```

## Testing WebRTC Leaks Programmatically

Create a test suite to verify protection:

```javascript
// webrtc-leak-test.js - Run in browser console
async function testWebRTCLeaks() {
  console.log("=== WebRTC Leak Detection Test ===\n");

  // Test 1: Check peerconnection availability
  const RTCPeerConnection = window.RTCPeerConnection ||
                            window.webkitRTCPeerConnection ||
                            window.mozRTCPeerConnection;

  if (!RTCPeerConnection) {
    console.log("✓ PASS: WebRTC disabled - RTCPeerConnection not available");
    return;
  }

  console.log("⚠ WARNING: WebRTC enabled - attempting leak detection\n");

  // Test 2: Attempt to extract IPs
  const ips = new Set();
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  pc.createDataChannel('');

  try {
    await pc.createOffer().then(offer => pc.setLocalDescription(offer));
  } catch(e) {
    console.log("✓ PASS: Cannot create offer - WebRTC disabled");
    return;
  }

  pc.onicecandidate = (ice) => {
    if (!ice || !ice.candidate) return;

    const candidate = ice.candidate.candidate;
    const ipMatch = candidate.match(/(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})/);

    if (ipMatch) {
      ips.add(ipMatch[1]);
      console.log(`✗ FAIL: IP leaked: ${ipMatch[1]}`);
    }
  };

  // Wait for candidates
  await new Promise(resolve => setTimeout(resolve, 2000));

  if (ips.size === 0) {
    console.log("✓ PASS: No IP addresses detected via WebRTC");
  } else {
    console.log(`\n⚠ CRITICAL: ${ips.size} IP(s) detected:`);
    ips.forEach(ip => console.log(`  - ${ip}`));
  }

  pc.close();
}

// Run test
testWebRTCLeaks();
```

## Monitoring WebRTC Status After Updates

Tor Browser updates may reset some user.js settings. Create a verification script:

```bash
#!/bin/bash
# verify-webrtc-disabled.sh - Check if WebRTC is disabled after Tor Browser update

TOR_PROFILE="$HOME/Library/Application Support/Tor Browser/Browser/profile.default"
PREFS_FILE="$TOR_PROFILE/prefs.js"

check_setting() {
  local setting=$1
  local expected=$2

  if grep -q "$setting.*$expected" "$PREFS_FILE" 2>/dev/null; then
    echo "✓ $setting = $expected"
    return 0
  else
    echo "✗ ALERT: $setting not set correctly"
    return 1
  fi
}

echo "Verifying WebRTC configuration..."
check_setting "media.peerconnection.enabled" "false"
check_setting "media.peerconnection.turn.disable" "true"
check_setting "media.peerconnection.use_document_iceservers" "false"

# Restore if settings missing
if [ $? -ne 0 ]; then
  echo "Restoring WebRTC security settings..."
  cp user.js "$TOR_PROFILE/"
  echo "user.js restored. Please restart Tor Browser."
fi
```

## Advanced: Virtual Machine Testing

For developers needing isolation during WebRTC testing:

```bash
# Create isolated test environment with network isolation
# VirtualBox example
VBoxManage createvm --name "tor-webrtc-test" --ostype Linux_64

# Configure VM with NAT network only (no host interface)
VBoxManage modifyvm "tor-webrtc-test" \
  --nic1 nat \
  --nictype1 virtio \
  --nat-network1 "NatNetwork"

# Install Tor Browser in VM, test WebRTC behavior
# Inspect network traffic from host with tcpdump
tcpdump -i vboxnet0 "port 3478 or port 5349" -n
```

## Verifying Your Protection

After implementing any of these methods, verify that WebRTC is properly disabled. Use multiple testing approaches:

First, check that `media.peerconnection.enabled` shows as false in about:config. Second, test with JavaScript-based leak detection tools. Third, check browser console output for any WebRTC-related errors or warnings when visiting websites that use the technology.

Use the test script provided above to verify across all browsers. Regular verification is important because browser updates can reset some settings. Maintain a checklist of your privacy configurations and review them after each Tor Browser update.

## Disabling WebRTC Across the Browser Ecosystem

If you use multiple browsers, WebRTC disabling strategies vary:

| Browser | Default Status | Disabling Method | Verification |
|---------|----------------|------------------|--------------|
| Tor Browser | Protected by design | Optional hardening | about:config |
| Firefox ESR | Enabled | about:config flag | Developer Tools |
| Chromium/Chrome | Enabled | chrome://flags or extensions | DevTools → Network |
| Brave | Protected by default | Settings → Privacy | Network inspection |

## Frequently Asked Questions

**How long does it take to disable webrtc leaks in tor browser?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How To Test Vpn For Webrtc Leaks Testing Guide](/privacy-tools-guide/how-to-test-vpn-for-webrtc-leaks--testing-guide/)
- [Verify VPN is Actually Working: DNS, WebRTC, IPv6 Leak Test](/privacy-tools-guide/how-to-verify-vpn-is-actually-working-dns-webrtc-ipv6-leak-test-guide/)
- [WebRTC Local IP Leak: How It Reveals Your Real Address](/privacy-tools-guide/webrtc-local-ip-leak-how-it-reveals-your-real-address/)
- [Brave Browser Crypto Features Disable Guide](/privacy-tools-guide/brave-browser-crypto-features-disable-guide/)
- [Best Browser for Tor Network 2026: A Technical Guide](/privacy-tools-guide/best-browser-for-tor-network-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
