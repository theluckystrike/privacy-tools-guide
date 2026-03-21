---
layout: default
title: "How to Disable Google AMP Tracking in Search Results Guide"
description: "Google AMP (Accelerated Mobile Pages) was introduced to speed up mobile web browsing, but it also creates significant privacy concerns. When you click an AMP"
date: 2026-03-16
author: theluckystrike
permalink: /how-to-disable-google-amp-tracking-in-search-results-guide/
categories: [guides]
tags: [privacy-tools-guide, privacy, google, amp, tracking, search]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

Google AMP (Accelerated Mobile Pages) was introduced to speed up mobile web browsing, but it also creates significant privacy concerns. When you click an AMP result in Google Search, your request routes through Google's servers, allowing Google to track your browsing activity across websites that would otherwise be independent. This guide covers practical methods to disable Google AMP tracking for developers and power users.

## How Google AMP Tracking Works

When you perform a Google Search on mobile or desktop, results often display AMP versions of web pages. Clicking these results sends your request through `google.com/amp` or `google-search` redirects, creating a detailed log of:

- The websites you visit
- Your search queries
- Timestamps and session data
- Device and browser information

Google's own documentation acknowledges this tracking, stating that AMP cache URLs provide analytics and performance data. Even if you visit a website directly later, the initial AMP click creates a persistent tracking record linked to your Google account.

## Method 1: Disable AMP in Google Search Settings

The simplest approach involves adjusting Google Search preferences:

1. Navigate to `google.com/search/settings`
2. Scroll to "Results" or "Search settings"
3. Look for "Accelerated Mobile Pages (AMP)" toggle
4. Disable AMP results

This method has limitations—Google may not offer this setting in all regions, and the setting can reset after clearing cookies or using incognito mode. Additionally, some search results may still redirect through AMP URLs even with this setting disabled.

## Method 2: Use Alternative Search Engines

Several privacy-focused search engines do not use AMP:

- **DuckDuckGo**: Does not use AMP or track search history
- **Startpage**: Returns original page versions without AMP
- **Brave Search**: Independent index without AMP integration
- **SearX**: Self-hosted meta search engine with no AMP

For developers building search interfaces, you can query these engines programmatically:

```bash
# Example: Query DuckDuckGo HTML (no JavaScript required)
curl -s "https://html.duckduckgo.com/html/?q=privacy+tools"
```

## Method 3: Browser Extensions for AMP Blocking

Several browser extensions strip AMP tracking:

### For Chrome/Brave/Edge:

**AMP Blocker** (Chrome Web Store) - Automatically blocks AMP domains and redirects to original content. The extension maintains a blocklist of known AMP trackers.

**Redirect AMP to HTML** - an userscript that converts AMP URLs to their canonical versions. Install with an userscript manager like Tampermonkey:

```javascript
// ==UserScript==
// @name         AMP Redirect
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Redirect AMP pages to normal pages
// @match        *://*.google.com/*
// @match        *://*.google.co.*/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';
    const url = window.location.href;
    if (url.includes('/amp/') || url.includes('amp=')) {
        const canonical = document.querySelector('link[rel="canonical"]');
        if (canonical) {
            window.location.href = canonical.href;
        }
    }
})();
```

### For Firefox:

**Privacy Badger** - EFF's tool automatically blocks known trackers including AMP-related tracking domains.

**uBlock Origin** - With proper filter lists, uBlock can block AMP tracking:

```
||google.com/amp$redirect=noop
||google.co.*/amp$redirect=noop
```

## Method 4: DNS-Level Blocking

For network-wide protection, you can block AMP tracking at the DNS level. This approach works for all devices on your network without individual configuration.

### Using Pi-hole:

Add these entries to your Pi-hole blocklist:

```
# Block Google AMP tracking domains
google.com.amp
www.google.com.amp
googlesyndication.com
googleadservices.com
doubleclick.net
```

### Custom DNS Providers:

Some privacy-focused DNS providers include tracker blocking:

```bash
# Test DNS configuration
dig +short A google.com.amp @1.1.1.1
dig +short A google.com.amp @9.9.9.9
```

If these domains resolve to blocked IPs, your DNS provider may already be filtering them.

## Method 5: Browser Configuration

Modern browsers offer settings to reduce AMP tracking:

### Firefox:

1. Navigate to `about:config`
2. Search for `privacy.trackingprotection`
3. Enable Enhanced Tracking Protection
4. Additionally, set `network.cookie.cookieBehavior` to `1` (block third-party cookies)

### Brave Browser:

Brave includes built-in AMP blocking:

1. Go to `brave://settings/shields`
2. Ensure "Fingerprinting" is set to "Strict"
3. Enable "Block Scripts" for additional protection

### Chrome (Limited Options):

Chrome's built-in privacy options are more restricted. Consider using the `--disable-google-top-replacement` flag:

```bash
# Linux/macOS launch option
google-chrome --disable-google-top-replacement

# Windows
chrome.exe --disable-google-top-replacement
```

## Method 6: Developer's Approach - AMP URL Detection

For developers building privacy tools, detecting and handling AMP URLs requires understanding their structure:

```python
import re
from urllib.parse import urlparse

def is_amp_url(url: str) -> bool:
    """Detect if a URL is an AMP version."""
    parsed = urlparse(url)
    
    # Check for /amp/ path
    if '/amp/' in parsed.path:
        return True
    
    # Check for google amp redirect parameters
    if 'amp' in parsed.query:
        return True
    
    # Check for google domain amp paths
    if 'google' in parsed.netloc and '/amp' in parsed.path:
        return True
    
    return False

def extract_canonical_url(amp_url: str) -> str:
    """Extract the original URL from an AMP URL."""
    # For URLs like https://example.com/amp/article
    # Should redirect to https://example.com/article
    pass  # Implementation depends on specific URL patterns
```

## Method 7: Use "NoAMP" Services

Several services provide "NoAMP" redirects:

- **noscript.com** - Browser extension that converts AMP to regular pages
- **amp2html** - Userscript that automatically redirects AMP pages

Install the amp2html userscript:

```javascript
// ==UserScript==
// @name         AMP2HTML
// @version      2.0
// @description  Convert AMP pages to regular HTML
// @match        *://*.google.com/amp/*
// @match        *://*.google.co.*/amp/*
// @run-at       document-start
// ==/UserScript==

document.addEventListener('DOMContentLoaded', () => {
    const canonical = document.querySelector('link[rel="canonical"]');
    if (canonical && canonical.href) {
        window.location.replace(canonical.href);
    }
});
```

## Verifying Your Protection

After implementing your chosen methods, verify AMP tracking is blocked:

1. Perform a Google Search for any topic
2. Click a result (preferably one with an AMP indicator)
3. Check the URL bar—if it shows the original domain (not google.com/amp), you're protected

You can also test programmatically:

```bash
# Check if AMP URLs resolve differently
curl -I "https://example.com/amp/article" 2>&1 | head -5
```

## Combining Methods for Maximum Protection

The most effective approach combines multiple methods:

- Use privacy-focused search engines as your default
- Install browser extensions for automatic AMP blocking
- Configure DNS-level blocking for network-wide protection
- Enable browser privacy settings

This layered approach ensures protection even if one method fails. For developers, implementing AMP detection in your own tools helps maintain privacy across all browsing activities.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
