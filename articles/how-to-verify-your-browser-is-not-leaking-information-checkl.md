---
layout: default
title: "Verify Your Browser is Not Leaking Information"
description: "A checklist for developers and power users to verify their browser is not leaking sensitive information. Includes code examples and testing methods"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-verify-your-browser-is-not-leaking-information-checkl/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Every browser has potential information leakage points that can expose your identity, location, or browsing habits to websites and third parties. For developers and power users, verifying that your browser is not leaking information requires testing multiple attack vectors systematically. This checklist provides actionable steps with code examples to audit your browser's privacy posture.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: WebRTC Leak Test

WebRTC (Web Real-Time Communication) can expose your real IP address even when using a VPN. This happens because WebRTC creates direct peer-to-peer connections that bypass VPN tunnels.

### How to Test for WebRTC Leaks

Create a simple test file:

```javascript
// Test for WebRTC IP leak
async function testWebRTCLeak() {
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  pc.createDataChannel('');

  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  return new Promise((resolve) => {
    pc.onicecandidate = (event) => {
      if (event.candidate) {
        console.log('IP Address Found:', event.candidate.candidate);
        pc.close();
        resolve(event.candidate.candidate);
      }
    };
  });
}

testWebRTCLeak();
```

If this code returns candidate information, your browser is leaking IP addresses through WebRTC. Firefox users can disable WebRTC by setting `media.peerconnection.enabled` to `false` in about:config. Chrome users should install an extension like WebRTC Leak Shield or use a browser with built-in WebRTC protection.

### Step 2: DNS Leak Test

Even with a VPN, your DNS requests might bypass the encrypted tunnel and go through your ISP's servers. This defeats the purpose of privacy-focused browsing.

### Testing DNS Leaks

Use dig or nslookup to verify which DNS server responds to your queries:

```bash
# Test which DNS server is resolving your queries
dig +short myip.opendns.com @resolver1.opendns.com
dig +short whoami.akamai.net @ns1-1.akamaitech.net

# Check your current DNS servers
cat /etc/resolv.conf  # Linux/macOS
ipconfig /all | findstr "DNS"  # Windows
```

For a more test, visit dnsleaktest.com or use the command-line tool:

```bash
# Using dnsleaktest CLI (if available)
dnsleaktest -t standard
```

If the DNS servers shown belong to your ISP rather than your VPN provider, you have a DNS leak. Fix this by configuring your system to use DNS-over-HTTPS (DoH) or DNS-over-TLS (DoT).

### Step 3: Canvas Fingerprinting Test

Canvas fingerprinting works by having your browser draw a hidden image and extracting the resulting pixel data. Different hardware and software configurations produce unique signatures.

### Testing Canvas Fingerprinting

Create a test to see if your canvas is readable:

```javascript
function getCanvasFingerprint() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  // Draw various elements that affect the fingerprint
  ctx.textBaseline = 'top';
  ctx.font = '14px Arial';
  ctx.fillStyle = '#f60';
  ctx.fillRect(125, 1, 62, 20);
  ctx.fillStyle = '#069';
  ctx.fillText('Privacy Test', 2, 15);
  ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
  ctx.fillText('Privacy Test', 4, 17);

  return canvas.toDataURL();
}

console.log('Canvas fingerprint:', getCanvasFingerprint());
```

If the function returns a consistent string across different websites, your browser has a unique canvas fingerprint. Firefox with `privacy.resistFingerprinting` enabled randomizes canvas output. Extensions like CanvasBlocker add noise to canvas readings.

### Step 4: WebGL Fingerprint Test

Similar to canvas fingerprinting, WebGL can expose your GPU renderer and vendor information, creating another unique identifier.

### Testing WebGL Information Leakage

```javascript
function getWebGLInfo() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

  if (!gl) return 'WebGL not supported';

  const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');

  return {
    vendor: gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL),
    renderer: gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL),
    version: gl.getParameter(gl.VERSION),
    shadingLanguage: gl.getParameter(gl.SHADING_LANGUAGE_VERSION)
  };
}

console.log('WebGL Info:', getWebGLInfo());
```

A privacy-hardened browser should either block this API or return generic values. Tor Browser, for example, standardizes WebGL output across all users.

### Step 5: User Agent and Navigator Properties

Your browser's user agent string and navigator properties reveal significant information about your system.

### Navigator Audit

```javascript
function auditNavigator() {
  const nav = navigator;

  return {
    userAgent: nav.userAgent,
    platform: nav.platform,
    language: nav.language,
    languages: nav.languages,
    hardwareConcurrency: nav.hardwareConcurrency,
    deviceMemory: nav.deviceMemory,
    doNotTrack: nav.doNotTrack,
    cookieEnabled: nav.cookieEnabled,
    pdfViewerEnabled: nav.pdfViewerEnabled,
    maxTouchPoints: nav.maxTouchPoints,
    timezone: Intl.DateTimeFormat().resolvedOptions().timeZone
  };
}

console.table(auditNavigator());
```

Compare this output across different browsers and incognito windows. A well-configured privacy browser should show consistent or generic values.

### Step 6: Configure Cookie and Storage Isolation Test

Verify that websites cannot access cookies and storage from other domains:

```javascript
function testStorageIsolation() {
  // Set a test cookie on current domain
  document.cookie = 'test_cookie=isolation_check; path=/; SameSite=Strict';

  // Check if we can read it back
  const canRead = document.cookie.includes('test_cookie');

  // Test localStorage
  try {
    localStorage.setItem('test_item', 'value');
    const localRead = localStorage.getItem('test_item') === 'value';
    localStorage.removeItem('test_item');
  } catch (e) {
    const localRead = false;
  }

  return { cookieAccess: canRead, localStorageAccess: localRead };
}

console.log('Storage Isolation:', testStorageIsolation());
```

### Step 7: Browser Privacy Checklist

Use this checklist to verify your browser configuration:

1. **WebRTC**: Disabled or leaking only VPN IP
2. **DNS**: Requests go through DoH/DoT or VPN tunnel
3. **Canvas**: Fingerprint is randomized or blocked
4. **WebGL**: Renderer info is masked or generic
5. **User Agent**: Does not reveal specific OS version
6. **Cookies**: Properly isolated between sites
7. **Third-party cookies**: Blocked
8. **Tracking protection**: Enabled and functioning
9. **Extensions**: Audited for excessive permissions
10. **Fingerprinting resistance**: Enabled in browser settings

### Step 8: Automation for Regular Auditing

For developers who want to automate these checks, create a Node.js script:

```javascript
const puppeteer = require('puppeteer');

async function auditBrowserPrivacy(url) {
  const browser = await puppeteer.launch({
    headless: true,
    args: ['--disable-web-security']
  });

  const page = await browser.newPage();
  await page.goto(url);

  // Inject and run all tests
  const results = await page.evaluate(() => {
    return {
      userAgent: navigator.userAgent,
      platform: navigator.platform,
      // Add other test results here
    };
  });

  await browser.close();
  return results;
}

auditBrowserPrivacy('https://example.com')
  .then(console.log)
  .catch(console.error);
```

Run this script periodically to track changes in your browser's privacy posture.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


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

- [Verify That Your VPN Is Actually Working and Not Leaking](/privacy-tools-guide/how-to-verify-that-your-vpn-is-actually-working-and-not-leaking/)
- [How to Verify Your VPN is Not Leaking DNS Requests in 2026](/privacy-tools-guide/how-to-verify-your-vpn-is-not-leaking-dns-requests/)
- [Google Nest Hub Data Collection](/privacy-tools-guide/google-nest-hub-data-collection-what-information-google-capt/)
- [Hotel Guest Privacy Rights What Information Hotels Can Share](/privacy-tools-guide/hotel-guest-privacy-rights-what-information-hotels-can-share/)
- [How To Remove Personal Information From Ai Training Datasets](/privacy-tools-guide/how-to-remove-personal-information-from-ai-training-datasets/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
