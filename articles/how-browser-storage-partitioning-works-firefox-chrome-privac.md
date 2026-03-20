---
layout: default
title: "How Browser Storage Partitioning Works: Firefox, Chrome."
description: "A technical guide to browser storage partitioning explaining how Firefox and Chrome isolate storage to prevent cross-site tracking, with code examples."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-browser-storage-partitioning-works-firefox-chrome-privac/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: false
---

{% raw %}

Modern browsers have adopted storage partitioning as a core defense mechanism against cross-site tracking. This article explains how storage partitioning works, examines implementation differences between Firefox and Chrome, and provides practical guidance for developers and power users who want to understand or test these privacy protections.

## What Is Storage Partitioning?

Storage partitioning is a browser mechanism that isolates website data based on the context in which a site is loaded. Without partitioning, websites can store data locally (cookies, localStorage, IndexedDB) and retrieve it even when the user visits other sites. This behavior enables cross-site tracking, where advertisers and analytics services build user profiles by sharing identifiers across multiple domains.

Storage partitioning changes this by scoping storage to a combination of the website origin and the top-level site. When you visit `example.com` embedded within `news-site.com` (via an iframe), the storage accessed is separate from what `example.com` would access when loaded as the top-level site or when embedded in a different parent site.

This means a third-party script embedded across multiple websites cannot correlate user activity because each embedding context provides a different storage partition.

## How Storage Partitioning Works

Browsers implement partitioning by modifying how they generate storage keys. Traditional storage uses the origin as the key:

```
Storage key = "https://example.com"
```

With partitioning, the key becomes a combination of the origin and the top-level site:

```
Storage key = ("https://example.com", "https://news-site.com")
```

This same principle applies across various storage APIs:

- **Cookies**: The `SameSite` attribute and cookie prefix behavior interact with partitioning
- **localStorage** and **sessionStorage**: Isolated per partition
- **IndexedDB**: Partitioned by top-level site
- **Cache API**: Also subject to partitioning in modern browsers

## Firefox Implementation

Firefox implements storage partitioning through its Enhanced Tracking Protection (ETP) feature, specifically "Total Cookie Protection." Firefox partitions all third-party storage by default in standard browsing mode.

You can verify Firefox's partitioning behavior by checking `about:config`:

```
browser.storage.partitioning.enabled = true
```

Firefox also provides detailed partitioning information in the Browser Toolbox. To access it:

1. Open `about:devtools-toolbox`
2. Select the storage inspector
3. Look for partition information in the storage tree

Here's a practical test you can run in Firefox's developer console:

```javascript
// Run in the main frame of example.com
localStorage.setItem('test-key', 'main-frame-value');

// Now run this in an iframe on a different domain
// You'll find a different localStorage instance
console.log(localStorage.getItem('test-key')); // null - different partition
```

Firefox's approach is aggressive—they partition all third-party storage automatically, which provides strong privacy but may break some legitimate cross-site functionality.

## Chrome Implementation

Chrome has been rolling out storage partitioning more gradually, referring to it as "Third-Party Cookie Deprecation" preparation, though the feature extends beyond cookies.

Chrome's partitioning applies to:

- LocalStorage and sessionStorage
- IndexedDB
- Cache API
- Cookie storage (with SomeParty cookie attribute)

You can observe Chrome's partitioning via the Application tab in DevTools. Select a storage entry and look for partition details in the side panel.

Chrome also provides a testing flag to force partitioning behavior:

```
chrome://flags/#storage-partitioning
```

To programmatically detect partitioning in Chrome:

```javascript
// Check if storage is partitioned
if (window.isolationSentinel) {
  // This property existed briefly to detect partitioning
  console.log('Partitioning may be active');
}

// Alternative: Check via console in DevTools
// Look for "Partition key" in Storage Inspector
```

Chrome's implementation allows some exceptions for specific use cases, particularly for legitimate cross-site functionality that requires shared state. The Partitioned cookie attribute (`Partitioned`) allows cookies to work in a partitioned context while maintaining privacy boundaries.

## Code Examples: Testing Partitioning

Here's a complete example demonstrating storage partitioning behavior:

```javascript
// script.js - Run this on site A (https://site-a.com)
function demonstratePartitioning() {
  // Set a value in localStorage
  localStorage.setItem('user-preference', 'dark-mode');
  
  // Set a cookie with Partitioned attribute (Firefox, some Chrome versions)
  document.cookie = 'session=abc123; SameSite=None; Secure; Partitioned';
  
  // Log current storage state
  console.log('LocalStorage value:', localStorage.getItem('user-preference'));
  console.log('Cookies:', document.cookie);
}

// Read storage on page load
window.addEventListener('DOMContentLoaded', demonstratePartitioning);
```

When `site-a.com` is embedded as a third-party on `site-b.com`, the storage accessed will be completely separate from storage accessed when `site-a.com` is the top-level site.

## Developer Considerations

For web developers, storage partitioning means:

1. **Third-party scripts may lose persistent state**: If your analytics or advertising scripts rely on localStorage or cookies set by your domain when embedded elsewhere, that data will no longer persist across different sites.

2. **Testing becomes more complex**: You need to test your site's behavior both as a top-level site and as embedded content, in different partitioning contexts.

3. **First-party solutions remain unaffected**: Storage set and read when your site is the top-level site continues to work normally.

4. **Consider using partitioning-aware APIs**: The Storage Access API allows embedded sites to request storage access, with browsers potentially granting permission based on user interaction.

```javascript
// Request storage access for embedded content
async function requestStorageAccess() {
  if (document.hasStorageAccess()) {
    return true;
  }
  
  try {
    const granted = await document.requestStorageAccess();
    return granted;
  } catch (error) {
    console.log('Storage access denied:', error);
    return false;
  }
}
```

## Power User Implications

For users concerned about privacy, storage partitioning provides significant protection:

- **Cross-site tracking is hindered**: Advertisers cannot use localStorage or cookies to track you across different websites
- **Fingerprinting becomes harder**: While fingerprinting can still occur through other methods, storage partitioning removes one major vector
- **Some site features may break**: Expect occasional issues with embedded content that relies on cross-site storage

You can verify your browser's partitioning status by:

1. **Firefox**: Check that Enhanced Tracking Protection is enabled in privacy settings
2. **Chrome**: Look for the "Third-party cookies" setting—selecting "Block third-party cookies" enables partitioning

## Conclusion

Storage partitioning represents a fundamental shift in how browsers handle web storage, creating isolation boundaries that prevent cross-site tracking while maintaining web functionality. Firefox implements this more aggressively, while Chrome has taken a more gradual approach with exceptions for certain use cases. Understanding these mechanisms helps developers build more robust web applications and empowers users to make informed privacy decisions.

For developers, the key takeaway is to avoid relying on third-party storage for persistent state, and to test applications in various embedding contexts. For users, enabling partitioning protections in browser settings provides meaningful privacy improvements with minimal downsides.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
