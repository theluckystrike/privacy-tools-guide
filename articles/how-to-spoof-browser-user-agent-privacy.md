---
---
layout: default
title: "How To Spoof Browser User Agent"
description: "Learn how to spoof browser user agent strings to enhance privacy, test web applications, and avoid fingerprinting. Practical techniques for developers"
date: 2026-03-15
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-spoof-browser-user-agent-privacy/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Every time your browser requests a webpage, it sends a User-Agent string that identifies your browser, operating system, and version. Websites use this information for analytics, device optimization, and sometimes for access control. However, this seemingly harmless header creates a fingerprint that trackers use to identify and follow users across the web.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand the User-Agent Header

The User-Agent header follows a standardized format:

```
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
```

This string reveals your browser vendor, version, operating system, and sometimes CPU architecture. Combined with other signals like screen resolution, installed fonts, and JavaScript capabilities, your User-Agent becomes part of a browser fingerprint that can track you even without cookies.

### Step 2: Browser Extensions for Quick Spoofing

The fastest way to modify your User-Agent without touching code is through browser extensions. Popular options include:

- **User-Agent Switcher** for Chrome and Firefox
- **SpoofDroid** for Android
- **Clystech User-Agent Switcher** for Safari

After installing an extension, you can select from preset User-Agent strings or create custom ones. This method works well for casual privacy improvements and quick testing, but power users should note that sophisticated trackers can still detect extension-based spoofing through JavaScript API checks.

### Step 3: Use Browser Developer Tools

Modern browsers include built-in options to override the User-Agent in their developer tools.

### Chrome and Edge

1. Open Developer Tools (F12 or Cmd+Option+I)
2. Click the three-dot menu → More tools → Network conditions
3. Uncheck "Use browser default" and enter your desired User-Agent string
4. Refresh the page to apply the change

This approach affects only the current tab and session, making it useful for testing how websites respond to different browsers without installing extensions.

### Firefox

Firefox requires a configuration change:

1. Type `about:config` in the address bar
2. Search for `general.useragent.override`
3. Create a new string preference with your desired User-Agent

This sets a persistent User-Agent for all Firefox sessions until you remove the preference.

### Step 4: Spoofing User-Agent in JavaScript

For web developers testing how their applications handle different User-Agents, JavaScript provides direct control:

```javascript
// Override the User-Agent for the current page
Object.defineProperty(navigator, 'userAgent', {
    value: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    writable: true
});

console.log(navigator.userAgent);
```

This JavaScript override affects only the page where you execute the code and doesn't persist across page reloads. For more persistent modifications, you can inject this code through an userscript manager like Tampermonkey or Violentmonkey.

### Step 5: Implement Programmatic Spoofing with Python

When building web scrapers or automated testing frameworks, Python offers well-maintained libraries for HTTP requests with custom User-Agent headers:

```python
import requests

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0'
}

response = requests.get('https://example.com', headers=headers)
print(response.text)
```

For more complex scenarios requiring JavaScript rendering, Selenium provides browser automation with User-Agent control:

```python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument('--user-agent=Mozilla/5.0 (Linux; Android 14; Pixel 8) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36')

driver = webdriver.Chrome(options=options)
driver.get('https://example.com')
```

### Step 6: Command-Line Tools for Privacy

Several command-line utilities let you browse or make requests with custom User-Agents:

### curl

```bash
curl -A "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1" https://example.com
```

The `-A` flag sets the User-Agent header directly.

### wget

```bash
wget --user-agent="Mozilla/5.0 (Linux; x86_64) Gecko/20100101" https://example.com
```

### htpx

The modern replacement for curl, `htpx`, offers similar functionality with improved syntax:

```bash
htpx https://example.com -H "User-Agent: Mozilla/5.0 (Windows NT 10.0)"
```

### Step 7: Limitations and Counter-Detection

While User-Agent spoofing provides basic privacy benefits, be aware of its limitations:

1. **Canvas and WebGL Fingerprinting**: Sophisticated trackers can still identify your browser through canvas rendering differences
2. **JavaScript API Consistency**: Modifying the User-Agent string doesn't change what `navigator.platform` or `navigator.appVersion` returns
3. **HTTP/2 and HTTP/3 Fingerprinting**: Modern protocols expose additional signals beyond headers

For stronger protection, combine User-Agent spoofing with other privacy measures like browser fingerprinting protection, privacy-focused browser extensions, and disabling JavaScript on untrusted sites.

### Step 8: Practical Recommendations

For developers testing cross-browser compatibility, use browser developer tools or automated testing frameworks like Playwright with custom User-Agent configurations. This approach provides accurate testing without permanent browser changes.

For privacy-conscious users, browser extensions offer the easiest entry point, though they should be combined with other privacy tools for better protection. Firefox with `privacy.resistFingerprinting` enabled provides solid baseline protection without additional configuration.

### Step 9: Browser Fingerprinting Defense

User-Agent spoofing addresses only one component of browser fingerprinting. A complete defense strategy requires multiple layers.

### Complete Firefox Hardening Configuration

Create a `user.js` configuration file in your Firefox profile that implements privacy:

```javascript
// Complete Firefox privacy hardening
user_pref("privacy.resistFingerprinting", true);
user_pref("privacy.resistFingerprinting.letterboxing", true);

// Disable User-Agent header to randomize it
user_pref("general.useragent.override", "");

// Canvas fingerprinting protection
user_pref("privacy.trackingprotection.enabled", true);
user_pref("privacy.trackingprotection.socialtracking.enabled", true);

// WebGL fingerprinting protection
user_pref("webgl.disabled", false); // Disable completely to stop leaks
user_pref("webgl.min_capability_mode", true);

// WebRTC leak protection
user_pref("media.peerconnection.enabled", false);

// Geolocation disabled by default
user_pref("geo.enabled", false);

// Disable device sensor APIs
user_pref("device.sensors.enabled", false);

// Disable battery status API
user_pref("dom.battery.enabled", false);

// Randomize timezone to prevent fingerprinting
user_pref("javascript.use_us_english_locale", true);
```

Download this configuration from privacy-focused repositories and place in your Firefox profile folder.

### Plugin and Codec Reporting

Websites detect installed plugins as a fingerprinting vector:

```javascript
// Check what plugins your browser reports
console.log('Plugins:', Array.from(navigator.plugins).map(p => p.name));

// Websites use this to fingerprint
// Chrome: No plugins (always empty)
// Firefox: Varies based on installed software
// Safari: Limited plugin list
```

Firefox with `privacy.resistFingerprinting` normalizes plugin reporting, while Chrome and Safari are harder to spoof.

### Font Enumeration Protection

Websites enumerate installed fonts to create fingerprints. Sophisticated tracking uses this vector:

```javascript
// Detecting available fonts
function detectInstalledFonts() {
  const fonts = [
    'Arial', 'Verdana', 'Times New Roman', 'Courier New',
    'Comic Sans MS', 'Trebuchet MS', 'Impact'
  ];

  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  return fonts.filter(font => {
    ctx.font = `12px ${font}`;
    const textWidth = ctx.measureText('test').width;
    // Return fonts that measure differently than default
  });
}
```

Mitigation involves using privacy-focused browsers that normalize font rendering or disabling web fonts entirely.

### Step 10: Browser Selection for Privacy

Different browsers provide different privacy baselines.

### Brave Browser: Balance of Privacy and Usability

Brave includes built-in protections without extensive configuration:

- Automatic HTTPS upgrade
- Tracker blocking by default
- WebRTC leak protection enabled
- Fingerprinting protection (randomizes Canvas, WebGL, Plugins)
- Custom User-Agent option in settings

Configuration for maximum privacy:

```
Settings → Privacy and security
- Block all cookies and site data
- Enable "Standard" fingerprinting protection
- Set User-Agent to random selection
```

### Firefox: Power User Flexibility

Firefox allows detailed configuration for users willing to read documentation:

```about:config``` settings for thorough privacy:

- `privacy.trackingprotection.enabled: true`
- `dom.event.clipboardevents.enabled: false` (prevent clipboard tracking)
- `dom.maxHardwareConcurrency: 2` (spoof core count)
- `dom.navigator.hardwareConcurrency: 2`

### Tor Browser: Maximum Anonymity

Tor Browser isolates fingerprinting through window letterboxing—maintaining consistent window size across all users, making fingerprinting less effective:

```
# Tor Browser default User-Agent
Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0) Gecko/20100101 Firefox/102.0
```

All Tor users report nearly identical User-Agents, making individual identification impossible.

### Safari: Minimal Configuration, Strong Defaults

Safari provides fingerprinting protection with limited user configuration:

- Built-in tracker blocking
- Intelligent Tracking Prevention (ITP)
- Randomized User-Agent (limited extent)

However, Safari's limited customization makes it difficult to implement advanced hardening.

### Step 11: Real-World Testing of User-Agent Effectiveness

Test your protection by visiting fingerprinting test sites.

### Browserleaks.com Full Testing

Visit browserleaks.com and run all tests:

1. User-Agent test (verify spoofed UA appears)
2. Canvas fingerprinting test (should appear randomized)
3. WebGL fingerprinting test (should appear randomized)
4. WebRTC IP leak test (should show VPN IP)
5. Device enumeration (should appear empty or generic)

Document results before and after hardening to verify effectiveness.

### AmIUnique.org Fingerprinting Assessment

AmIUnique calculates how unique your browser fingerprint is:

```
Expected uniqueness: 1 in 1 million
(Indicates strong protection - high uniqueness defeats tracking)
```

Aim for "1 in several thousand" or higher uniqueness. Very low numbers indicate you're easily identifiable.

### Step 12: User-Agent Randomization Strategies

Rather than static spoofing, randomization improves privacy:

### Mulvad's Browser Approach

Mullvad's free browser randomizes User-Agent with every page load:

```javascript
// Simplified version of what Mullvad does
function randomizeUserAgent() {
 const browserVariants = [
 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0',
 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) Chrome/120.0.0.0',
 'Mozilla/5.0 (X11; Linux x86_64) Chrome/120.0.0.0'
 ];

 return browserVariants[Math.floor(Math.random() * browserVariants.length)];
}
```

This makes cross-site tracking harder because your User-Agent differs on each visit.

### Randomized User-Agent Chrome Extension

Create a simple extension that randomizes User-Agent:

```json
{
 "manifest_version": 3,
 "name": "User-Agent Randomizer",
 "permissions": ["webRequest", "webRequestBlocking"],
 "host_permissions": ["<all_urls>"],

 "background": {
 "service_worker": "background.js"
 }
}
```

```javascript
// background.js
const userAgents = [
 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0',
 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) Safari/537.36',
 'Mozilla/5.0 (X11; Linux x86_64; rv:121.0) Gecko/20100101 Firefox/121.0'
];

chrome.webRequest.onBeforeSendHeaders.addListener(
 (details) => {
 return {
 requestHeaders: details.requestHeaders.map(header => {
 if (header.name === 'User-Agent') {
 return {
 name: 'User-Agent',
 value: userAgents[Math.floor(Math.random() * userAgents.length)]
 };
 }
 return header;
 })
 };
 },
 { urls: ['<all_urls>'] },
 ['blocking', 'requestHeaders']
);
```

### Step 13: Practical Recommendations for Different User Types

### Web Developers

For developers testing cross-browser compatibility:

1. Use developer tools built-in User-Agent overrides for testing
2. Use Playwright with custom User-Agent configurations:

```javascript
const browser = await chromium.launch();
const context = await browser.newContext({
 userAgent: 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)'
});
const page = await context.newPage();
```

3. Test against Firefox and Chrome—most users employ one of these

### Privacy-Conscious Individuals

For maximum privacy:

1. Use Tor Browser for all sensitive browsing
2. Use Brave Browser for everyday use with hardening
3. Avoid Chrome/Chromium (poor privacy defaults)
4. Combine User-Agent spoofing with other privacy measures

### Enterprise Users

For organizational deployments:

1. Configure User-Agent through group policy for consistency
2. Avoid relying solely on User-Agent spoofing (use VPN + proxy)
3. Document expected User-Agents for infrastructure
4. Don't spoof User-Agent on internal systems (breaks authentication)

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to spoof browser user agent?**

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

- [How to Check What Your Browser Reveals: A Developer Guide](/privacy-tools-guide/how-to-check-what-your-browser-reveals/)
- [Verify Your Browser is Not Leaking Information](/privacy-tools-guide/how-to-verify-your-browser-is-not-leaking-information-checkl/)
- [Tor Browser vs Epic Privacy Browser Comparison](/privacy-tools-guide/tor-browser-vs-epic-privacy-browser-comparison/)
- [How To Check Your Browser Fingerprint Uniqueness Score](/privacy-tools-guide/how-to-check-your-browser-fingerprint-uniqueness-score-onlin/)
- [How to Test if Your Anti-Fingerprinting Setup Actually](/privacy-tools-guide/how-to-test-if-your-anti-fingerprinting-setup-actually-works/)
```

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
```
```
