---
layout: default
title: "How Browser Fingerprinting Works Explained"
description: "A developer-focused explanation of browser fingerprinting techniques, with practical code examples showing how trackers identify users without cookies"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /how-browser-fingerprinting-works-explained/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Browser fingerprinting tracks you by combining your screen resolution, installed fonts, GPU model, audio processing quirks, and dozens of other browser attributes into a unique identifier — no cookies required. Because these signals come from APIs browsers must expose for normal functionality, fingerprints are far harder to block than cookies and can follow you across sessions, sites, and even different browsers on the same device. Below is a technical breakdown of each fingerprinting vector, with working code examples and practical defenses.

## What Makes Fingerprinting Possible

Every browser exposes a range of information when visiting websites. This includes installed fonts, screen resolution, timezone, hardware concurrency, and installed plugins. When combined, these attributes create a unique signature that can identify users across sessions and even across different browsers on the same device.

The fundamental challenge is that many of these attributes serve legitimate purposes. Websites need to know screen dimensions for responsive design, timezone for localized content, and hardware capabilities for performance optimization. Fingerprinting exploits this legitimate data access for tracking purposes.

## Core Fingerprinting Techniques

### Canvas Fingerprinting

HTML5 Canvas allows websites to draw graphics programmatically. Different browsers and operating systems render graphics slightly differently due to variations in graphics drivers, font rendering, and anti-aliasing. By drawing a hidden image and extracting its data hash, trackers create a unique identifier:

```javascript
function getCanvasFingerprint() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  // Draw various elements that produce rendering differences
  ctx.textBaseline = 'top';
  ctx.font = '14px Arial';
  ctx.fillText('Hello World', 2, 2);
  ctx.fillStyle = '#f60';
  ctx.fillRect(125, 1, 62, 20);
  ctx.fillStyle = '#069';
  ctx.fillText('Hello World', 4, 17);
  ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
  ctx.fillText('Hello World', 7, 31);

  // Extract the canvas data as a base64 string
  const dataURL = canvas.toDataURL();

  // Hash the data to create a fingerprint
  let hash = 0;
  for (let i = 0; i < dataURL.length; i++) {
    const char = dataURL.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash;
  }

  return hash.toString(16);
}

console.log('Canvas fingerprint:', getCanvasFingerprint());
```

The resulting hash varies across devices even for the same image. Graphics cards, drivers, and operating systems all contribute to subtle rendering differences that fingerprinting can detect.

### WebGL Fingerprinting

WebGL provides low-level graphics capabilities that expose even more hardware information. Beyond canvas-like rendering differences, WebGL can report the graphics card vendor and renderer:

```javascript
function getWebGLFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

  if (!gl) {
    return 'WebGL not supported';
  }

  const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');

  if (!debugInfo) {
    return 'Debug info not available';
  }

  const vendor = gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL);
  const renderer = gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL);

  return {
    vendor: vendor,
    renderer: renderer
  };
}

console.log('WebGL fingerprint:', getWebGLFingerprint());
```

This reveals the exact graphics card model, which is highly identifying when combined with other attributes.

### Font Detection

Websites cannot directly list installed fonts, but they can detect fonts through a technique comparing element widths. By measuring text width with a specific font versus a fallback font, scripts determine which fonts are present:

```javascript
function detectFont(fontName) {
  const baseFonts = ['monospace', 'sans-serif', 'serif'];
  const testString = 'mmmmmmmmmmlli';
  const testSize = '72px';

  const span = document.createElement('span');
  span.style.fontSize = testSize;
  span.innerHTML = testString;
  span.style.visibility = 'hidden';
  document.body.appendChild(span);

  // Measure with fallback font
  const baseWidths = {};
  for (const baseFont of baseFonts) {
    span.style.fontFamily = baseFont;
    baseWidths[baseFont] = span.offsetWidth;
  }

  // Measure with target font
  span.style.fontFamily = `"${fontName}", ${baseFonts[0]}`;
  const targetWidth = span.offsetWidth;

  document.body.removeChild(span);

  // If width differs from fallback, font exists
  for (const baseFont of baseFonts) {
    if (targetWidth !== baseWidths[baseFont]) {
      return true;
    }
  }

  return false;
}

// Check for specific fonts
const fonts = ['Helvetica', 'Courier New', 'Times New Roman', 'Fira Code', 'SF Pro'];
const detectedFonts = fonts.filter(detectFont);
console.log('Detected fonts:', detectedFonts);
```

Font detection is particularly powerful because users often install custom fonts, creating unique combinations that distinguish their browser from others.

### Hardware and API Fingerprinting

Modern browsers expose numerous APIs that reveal hardware capabilities:

```javascript
function getHardwareFingerprint() {
  const fp = {
    // Screen and window dimensions
    screenWidth: screen.width,
    screenHeight: screen.height,
    colorDepth: screen.colorDepth,
    pixelRatio: window.devicePixelRatio,

    // Hardware concurrency (CPU cores)
    hardwareConcurrency: navigator.hardwareConcurrency || 'unknown',

    // Device memory (if available)
    deviceMemory: navigator.deviceMemory || 'unknown',

    // Platform and user agent
    platform: navigator.platform,
    userAgent: navigator.userAgent,

    // Timezone
    timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
    timezoneOffset: new Date().getTimezoneOffset(),

    // Touch support
    maxTouchPoints: navigator.maxTouchPoints || 0,
    touchSupport: 'ontouchstart' in window,

    // Language
    language: navigator.language,
    languages: navigator.languages,

    // Do Not Track setting
    doNotTrack: navigator.doNotTrack || 'unspecified'
  };

  return fp;
}

console.log('Hardware fingerprint:', JSON.stringify(getHardwareFingerprint(), null, 2));
```

## Advanced Fingerprinting Vectors

### Audio Context Fingerprinting

The Web Audio API can fingerprint audio processing differences:

```javascript
function getAudioFingerprint() {
  const audioContext = new (window.AudioContext || window.webkitAudioContext)();

  // Create an oscillator and destination
  const oscillator = audioContext.createOscillator();
  const analyser = audioContext.createAnalyser();
  const gain = audioContext.createGain();
  const scriptProcessor = audioContext.createScriptProcessor(4096, 1, 1);

  // Configure oscillator
  oscillator.type = 'triangle';
  oscillator.frequency.setValueAtTime(10000, audioContext.currentTime);

  // Connect nodes
  gain.gain.setValueAtTime(0, audioContext.currentTime);
  gain.gain.linearRampToValueAtTime(0, audioContext.currentTime + 0.01);
  oscillator.connect(gain);
  gain.connect(analyser);
  analyser.connect(scriptProcessor);
  scriptProcessor.connect(audioContext.destination);

  // Process audio and get result
  const fingerprint = scriptProcessor.onaudioprocess = function(e) {
    const data = e.inputBuffer.getChannelData(0);
    let sum = 0;
    for (let i = 0; i < data.length; i++) {
      sum += Math.abs(data[i]);
    }
    return sum;
  };

  oscillator.start();

  // Return hash of audio processing
  return fingerprint.toString();
}
```

### Battery Status API

If available, the Battery API provides unique information:

```javascript
async function getBatteryFingerprint() {
  if (!navigator.getBattery) {
    return 'Battery API not available';
  }

  const battery = await navigator.getBattery();

  return {
    charging: battery.charging,
    chargingTime: battery.chargingTime,
    dischargingTime: battery.dischargingTime,
    level: battery.level
  };
}
```

## Fingerprinting Resistance

Developers and users can reduce fingerprinting effectiveness through various approaches:

**For users:**
- Use browsers with built-in fingerprinting protection (Firefox with Enhanced Tracking Protection, Brave Browser)
- Disable JavaScript for untrusted sites (extreme measure, breaks many sites)
- Use privacy-focused extensions that block known fingerprinting scripts
- Avoid customizing browser appearance or installing unusual fonts

**For developers:**
- Implement feature policy restrictions to limit API exposure
- Use normalized or randomized values for fingerprintable attributes
- Employ Content Security Policy to block third-party tracking scripts
- Consider the privacy implications before using any API that exposes user information

## The Privacy Implications

Browser fingerprinting operates by extracting information the browser must provide for normal functionality—unlike cookies, it does not require browser cooperation. This makes it harder to block without breaking websites.

The same APIs used for fingerprinting often have legitimate uses, so the challenge for developers is building applications that function properly while minimizing unnecessary information disclosure.

Browser selection matters for users concerned about tracking. Some browsers actively randomize fingerprintable values, but this protection only works when enough users share similar fingerprints—the more unique a configuration, the easier it is to track regardless of browser features.


## Related Articles

- [Browser Fingerprinting How It Works and How to Prevent It](/privacy-tools-guide/browser-fingerprinting-how-it-works-and-how-to-prevent-it-guide/)
- [How to Test if Your Anti-Fingerprinting Setup Actually Works](/privacy-tools-guide/how-to-test-if-your-anti-fingerprinting-setup-actually-works/)
- [Tor Circuit: How It Works and Visualization Explained](/privacy-tools-guide/tor-circuit-how-it-works-visualization-explained/)
- [Client Hints API: The New Chrome Tracking Vector Explained](/privacy-tools-guide/client-hints-api-fingerprinting-new-chrome-tracking-vector-e/)
- [Browser First-Party Isolation: What It Does and How It Works](/privacy-tools-guide/browser-first-party-isolation-what-it-does/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
