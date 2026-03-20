---

layout: default
title: "How to Spoof Browser User Agent for Privacy: A Practical."
description: "Learn how to spoof browser user agent strings to enhance privacy, test web applications, and avoid fingerprinting. Practical techniques for developers."
date: 2026-03-15
author: theluckystrike
permalink: /how-to-spoof-browser-user-agent-privacy/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Every time your browser requests a webpage, it sends a User-Agent string that identifies your browser, operating system, and version. Websites use this information for analytics, device optimization, and sometimes for access control. However, this seemingly harmless header creates a fingerprint that trackers use to identify and follow users across the web.

## Understanding the User-Agent Header

The User-Agent header follows a standardized format:

```
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
```

This string reveals your browser vendor, version, operating system, and sometimes CPU architecture. Combined with other signals like screen resolution, installed fonts, and JavaScript capabilities, your User-Agent becomes part of a browser fingerprint that can track you even without cookies.

## Browser Extensions for Quick Spoofing

The fastest way to modify your User-Agent without touching code is through browser extensions. Popular options include:

- **User-Agent Switcher** for Chrome and Firefox
- **SpoofDroid** for Android
- **Clystech User-Agent Switcher** for Safari

After installing an extension, you can select from preset User-Agent strings or create custom ones. This method works well for casual privacy improvements and quick testing, but power users should note that sophisticated trackers can still detect extension-based spoofing through JavaScript API checks.

## Using Browser Developer Tools

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

## Spoofing User-Agent in JavaScript

For web developers testing how their applications handle different User-Agents, JavaScript provides direct control:

```javascript
// Override the User-Agent for the current page
Object.defineProperty(navigator, 'userAgent', {
    value: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    writable: true
});

console.log(navigator.userAgent);
```

This JavaScript override affects only the page where you execute the code and doesn't persist across page reloads. For more persistent modifications, you can inject this code through a userscript manager like Tampermonkey or Violentmonkey.

## Programmatic Spoofing with Python

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

## Command-Line Tools for Privacy

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

## Limitations and Counter-Detection

While User-Agent spoofing provides basic privacy benefits, be aware of its limitations:

1. **Canvas and WebGL Fingerprinting**: Sophisticated trackers can still identify your browser through canvas rendering differences
2. **JavaScript API Consistency**: Modifying the User-Agent string doesn't change what `navigator.platform` or `navigator.appVersion` returns
3. **HTTP/2 and HTTP/3 Fingerprinting**: Modern protocols expose additional signals beyond headers

For stronger protection, combine User-Agent spoofing with other privacy measures like browser fingerprinting protection, privacy-focused browser extensions, and disabling JavaScript on untrusted sites.

## Practical Recommendations

For developers testing cross-browser compatibility, use browser developer tools or automated testing frameworks like Playwright with custom User-Agent configurations. This approach provides accurate testing without permanent browser changes.

For privacy-conscious users, browser extensions offer the easiest entry point, though they should be combined with other privacy tools for better protection. Firefox with `privacy.resistFingerprinting` enabled provides solid baseline protection without additional configuration.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
