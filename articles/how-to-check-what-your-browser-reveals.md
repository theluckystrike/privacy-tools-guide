---
layout: default
title: "How to Check What Your Browser Reveals: A Developer Guide"
description: "Learn how to check what your browser reveals to websites. Practical methods, code examples, and tools for developers to audit browser fingerprinting."
date: 2026-03-15
author: theluckystrike
permalink: /how-to-check-what-your-browser-reveals/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Every time you visit a website, your browser hands over a surprising amount of information. Beyond basic details like your IP address and user agent, websites can extract screen resolution, installed fonts, GPU renderer, timezone, language preferences, and hundreds of other signals. Combined, these create a fingerprint that can identify you across sessions without cookies.

For developers and power users, understanding what your browser reveals is the first step toward hardening your privacy posture. This guide covers practical methods to audit your browser's digital footprint with working code examples.

## What Your Browser Exposes by Default

Modern browsers expose numerous APIs for legitimate functionality, but these same APIs enable fingerprinting. The key categories include:

**Basic Navigator Properties**
- User Agent string
- Platform and OS version
- Language and languages accepted
- Hardware concurrency (CPU cores)
- Device memory (if available)

**Screen and Display**
- Screen resolution and color depth
- Available screen space
- Pixel ratio

**Hardware Features**
- GPU renderer and vendor
- Audio context fingerprint
- WebGL renderer info

**Browser State**
- Cookies enabled
- Do Not Track setting
- Timezone
- Connection type

## Using the Navigator API

The simplest way to see what your browser reveals is directly querying the Navigator API. Create a test HTML file:

```html
<!DOCTYPE html>
<html>
<head><title>Browser Fingerprint Audit</title></head>
<body>
<h1>Navigator Properties</h1>
<pre id="output"></pre>
<script>
const nav = navigator;
const properties = [
  'userAgent', 'platform', 'language', 'languages',
  'hardwareConcurrency', 'deviceMemory', 'cookieEnabled',
  'doNotTrack', 'timezone'
];

let output = properties.map(prop => {
  const value = nav[prop] !== undefined ? nav[prop] : 'Not available';
  return `${prop}: ${JSON.stringify(value)}`;
}).join('\n');

document.getElementById('output').textContent = output;
</script>
</body>
</html>
```

Open this file in your browser and examine the output. You'll likely see your exact browser version, operating system, and language preferences exposed.

## Querying Screen Properties

Screen information adds another fingerprinting vector. Query the Screen API:

```javascript
const screenInfo = {
  width: screen.width,
  height: screen.height,
  availWidth: screen.availWidth,
  availHeight: screen.availHeight,
  colorDepth: screen.colorDepth,
  pixelRatio: window.devicePixelRatio
};

console.table(screenInfo);
```

These values reveal your monitor resolution, whether taskbars are visible, and your display configuration. Multiple monitors or unusual resolutions make you more identifiable.

## Extracting GPU Information

Graphics card details are particularly valuable for fingerprinting. Use WebGL:

```javascript
function getGpuInfo() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
  
  if (!gl) return { error: 'WebGL not supported' };
  
  const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
  
  if (!debugInfo) return { error: 'Debug info not available' };
  
  return {
    vendor: gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL),
    renderer: gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL)
  };
}

console.log(getGpuInfo());
```

This exposes your exact graphics card model. Combined with your CPU and screen resolution, this creates a highly unique fingerprint.

## Canvas Fingerprinting Test

Canvas fingerprinting works by drawing a hidden image and extracting its hash. Different browsers and GPUs render slightly differently, creating a unique signature:

```javascript
function getCanvasFingerprint() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  
  canvas.width = 200;
  canvas.height = 50;
  
  ctx.textBaseline = 'top';
  ctx.font = '14px Arial';
  ctx.fillStyle = '#f60';
  ctx.fillRect(125, 1, 62, 20);
  ctx.fillStyle = '#069';
  ctx.fillText('Fingerprint Test', 2, 15);
  ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
  ctx.fillText('Fingerprint Test', 4, 17);
  
  return canvas.toDataURL();
}

const fingerprint = getCanvasFingerprint();
console.log('Canvas hash:', fingerprint.slice(0, 50) + '...');
```

The resulting data URL differs between browsers and devices, even when running the same code.

## Font Detection

Installed fonts provide another fingerprinting vector. This technique works by measuring text width with different font families:

```javascript
function detectFonts(baseFonts, testFonts) {
  const span = document.createElement('span');
  span.innerHTML = 'mmmmmmmmmmlli';
  span.style.fontSize = '72px';
  span.style.position = 'absolute';
  span.style.left = '-9999px';
  document.body.appendChild(span);
  
  const detected = [];
  
  baseFonts.forEach(font => {
    span.style.fontFamily = font;
    const baseWidth = span.offsetWidth;
    
    testFonts.forEach(testFont => {
      span.style.fontFamily = `'${testFont}', ${font}`;
      if (span.offsetWidth !== baseWidth) {
        detected.push(testFont);
      }
    });
  });
  
  document.body.removeChild(span);
  return detected;
}

const baseFonts = ['monospace', 'sans-serif', 'serif'];
const testFonts = ['Arial', 'Helvetica', 'Times New Roman', 'Courier New', 'Verdana', 'Georgia'];

console.log('Detected fonts:', detectFonts(baseFonts, testFonts));
```

This reveals which fonts you have installed, and the specific combination is highly identifying.

## WebRTC Leak Detection

WebRTC can expose your real IP address even behind a VPN:

```javascript
async function checkWebRTC() {
  const rtc = new RTCPeerConnection({ iceServers: [] });
  
  return new Promise((resolve) => {
    rtc.createDataChannel('');
    
    rtc.onicecandidate = (e) => {
      if (e.candidate) {
        const candidate = e.candidate.candidate;
        const ipMatch = candidate.match(/(\d{1,3}\.){3}\d{1,3}/);
        if (ipMatch) {
          resolve({ ip: ipMatch[0], full: candidate });
          rtc.close();
        }
      }
    };
    
    rtc.createOffer().then(o => rtc.setLocalDescription(o));
    
    setTimeout(() => {
      resolve({ error: 'No WebRTC leak detected within timeout' });
      rtc.close();
    }, 2000);
  });
}

checkWebRTC().then(result => console.log('WebRTC result:', result));
```

If this returns an IP address that differs from your VPN's IP, you have a WebRTC leak.

## Using Existing Fingerprinting Test Sites

While building your own tests provides the deepest understanding, several established tools audit browser fingerprinting:

**Panopticlick/EFF Cover Your Tracks**
Tests how unique your browser is based on exposed attributes. Higher uniqueness means easier tracking.

**AmIUnique**
European project analyzing fingerprint data. Provides detailed breakdown of what makes your browser identifiable.

**BrowserLeaks**
Comprehensive fingerprinting tests including canvas, WebGL, audio, and WebRTC vectors.

These tools compare your fingerprint against a database to calculate your uniqueness score.

## Reducing Your Digital Footprint

After auditing what your browser reveals, consider these hardening steps:

**Browser Selection**
Privacy-focused browsers like Firefox with arkenfox configuration, Brave, or Tor Browser block many fingerprinting vectors by default.

**Disable JavaScript**
Tor Browser can disable JavaScript entirely, though many sites will break. Use NoScript extension for granular control.

**Resistance Features**
Firefox 128+ includes "resistFingerprinting" which normalizes many exposed values. Enable in `about:config`:

```
privacy.resistFingerprinting = true
```

**WebRTC Blocking**
Disable WebRTC in browser settings or use extensions that block the leak.

## Conclusion

Your browser reveals substantially more information than most users realize. By running the code examples above, you can audit exactly what websites see. The Navigator API exposes basic properties, WebGL reveals your GPU, canvas rendering creates unique hashes, and font detection exposes your installed typefaces.

Understanding these fingerprinting vectors is essential for developers building privacy-conscious applications and for users who want to minimize their digital footprint. The methods shown here give you the tools to test, audit, and ultimately reduce what your browser reveals.


## Related Reading

- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
