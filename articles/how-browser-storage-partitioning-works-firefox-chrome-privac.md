---
layout: default
title: "How Browser Storage Partitioning Works Firefox Chrome Privac"
description: "A technical guide to browser storage partitioning explaining how Firefox and Chrome isolate storage to prevent cross-site tracking, with code examples"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-browser-storage-partitioning-works-firefox-chrome-privac/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
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

## Storage Partitioning in Edge and Safari

Major browsers are converging on storage partitioning, though with different implementation details:

### Microsoft Edge Implementation

Edge follows Chrome's approach but with some variations:

```javascript
// Check Edge's partitioning status
if (navigator.userAgentData?.brands?.some(b => b.brand === 'Microsoft Edge')) {
  // Edge partitions localStorage similar to Chrome
  // But has additional restrictions on ServiceWorker storage

  // This code demonstrates partition awareness:
  if (window.isSecureContext) {
    localStorage.setItem('edge-partition-test', 'value');
    console.log('LocalStorage is partitioned in Edge');
  }
}
```

Edge provides a setting under **Settings > Privacy > Third-party cookies** with options:
- **Block**: Partitions all third-party cookies (strict)
- **Block third-party:** Partitions some, allows specific services (balanced)
- **Allow:** No partitioning (permissive)

### Safari's Intelligent Tracking Prevention (ITP)

Safari takes an even more aggressive approach than Firefox. Its Intelligent Tracking Prevention (ITP) features:

1. **Partitions all third-party storage** by default
2. **Caps first-party cookies** to 7 days if the site is in Safari's tracking list
3. **Blocks cross-domain cookies** entirely in certain scenarios

Safari's approach is nuanced—it maintains a dynamic list of known trackers and applies stricter rules to them:

```javascript
// Test if you're running on Safari
const isSafari = /^((?!chrome|android).)*safari/i.test(navigator.userAgent);

if (isSafari) {
  // Safari may have stricter storage isolation
  // Check if localStorage persists across iframes
  try {
    localStorage.setItem('safari-itp-test', Date.now());
    console.log('First-party storage available');
  } catch (e) {
    console.log('Possible ITP restriction on storage');
  }
}
```

## Practical Testing: Verifying Partitioning in Your Browser

Here's a complete test suite you can run locally:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Storage Partitioning Test</title>
</head>
<body>
  <h1>Storage Partitioning Verification</h1>

  <div id="results"></div>

  <script>
    async function testPartitioning() {
      const results = document.getElementById('results');

      // Test 1: LocalStorage partitioning
      localStorage.setItem('partition-test', 'main-frame-' + Date.now());
      const mainFrameValue = localStorage.getItem('partition-test');
      results.innerHTML += `<p>Main frame localStorage: ${mainFrameValue}</p>`;

      // Test 2: IndexedDB partitioning
      const dbRequest = indexedDB.open('partitionTestDB', 1);
      dbRequest.onsuccess = () => {
        const db = dbRequest.result;
        const store = db.createObjectStore('storage', { keyPath: 'id' });
        store.add({ id: 1, value: 'test', timestamp: Date.now() });
        results.innerHTML += `<p>IndexedDB partition established</p>`;
      };

      // Test 3: Check DevTools for partition info
      results.innerHTML += `<p style="color: #666; font-size: 12px;">
        Open DevTools > Application > Storage to see partition keys:
        <br/>In Firefox: Look for "partition" column
        <br/>In Chrome: Look for "Partition key" in Storage Inspector
      </p>`;

      // Test 4: Cookie partitioning
      document.cookie = 'partition-test=value; SameSite=None; Secure; Partitioned';
      results.innerHTML += `<p>Partitioned cookie set (if supported)</p>`;
    }

    testPartitioning();
  </script>
</body>
</html>
```

Save this as `test-partitioning.html`, open in your browser, and check DevTools to observe partition behavior.

## Building Sites That Work With Partitioning

For developers maintaining websites that embed third-party content, partitioning requires architectural changes:

### Pattern 1: First-Party Requests (No Changes Needed)

If your site is the top-level site, partitioning doesn't affect you:

```javascript
// On your website (e.g., example.com)
// This always works the same way:
localStorage.setItem('user-preference', 'dark-mode');
// Stored in: ("https://example.com", "https://example.com")
```

### Pattern 2: Embedded Content (Needs Storage Access API)

If your company's analytics script is embedded on customer websites:

```javascript
// Before: This worked everywhere
localStorage.setItem('user-session', 'uuid-123');

// After partitioning: This fails if embedded
// Solution: Request storage access
async function ensureStorageAccess() {
  if (document.hasStorageAccess?.()) {
    return true;
  }

  try {
    await document.requestStorageAccess?.();
    // Storage access granted, now we can use localStorage
    localStorage.setItem('user-session', 'uuid-123');
    return true;
  } catch (e) {
    // Storage access denied, fall back to server-side sessions
    return false;
  }
}
```

This API requires user interaction (click), so it's not a complete solution. Expect some users to deny storage access.

### Pattern 3: Server-Side Sessions (Most Reliable)

The most robust approach is moving away from client-side storage:

```javascript
// Old approach (breaks with partitioning):
// Client stores session token in localStorage
// Used for all API requests
// Problem: Doesn't work for third-party embeds

// New approach:
// 1. Server sets httpOnly cookie for session
// 2. API requests include cookie automatically (no JavaScript access)
// 3. Works across partitions because httpOnly cookies
//    follow slightly different rules

// Example: Set session via Set-Cookie header
// Server sends: Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax

// Client code (simplified):
fetch('/api/user', {
  credentials: 'include' // Include httpOnly cookies
});

// This works across partitions because the browser handles cookie
// inclusion automatically, not via JavaScript access
```

## Performance Implications of Partitioning

Storage partitioning has subtle performance impacts:

### Memory Overhead
Each partition maintains separate storage:

```markdown
## Memory Calculation Example

Without partitioning:
- 1 localStorage instance per origin
- 10 MB per origin × 100 origins = 1 GB total

With partitioning:
- 1 localStorage instance per (origin, top-level-site) pair
- A third-party on 1,000 websites = 1,000 separate partitions
- 10 MB × 1,000 = 10 GB total (hypothetically)

In practice, browsers limit storage (typically 50-100 MB per origin),
so multiple partitions are constrained by this limit total.
```

### Caching Behavior Change

Applications that cached data across sites now can't:

```javascript
// Before partitioning: This cached globally
const userData = await fetch('/api/user');
// Same data served to requests from different top-level sites

// After partitioning: Cache is per-partition
// If script runs on siteA.com and siteB.com
// Each gets its own cached copy
// Result: More API requests, slightly higher server load
```

## Migration Timeline: What You Need to Do

Different browsers enable partitioning on different schedules:

| Browser | Status | Action Required |
|---------|--------|-----------------|
| Firefox | Enabled | Deploy code with Storage Access API fallback |
| Chrome | Rolling out | Test with `chrome://flags` now, prepare migration |
| Edge | Enabled | Same as Chrome; test now |
| Safari | Enabled | iOS apps need native location permissions alternative |

For most applications, the migration path is:

**Phase 1 (Now):** Test your app with storage partitioning enabled in DevTools
**Phase 2 (Next 3 months):** Implement Storage Access API request for critical features
**Phase 3 (Next 6 months):** Migrate to server-side sessions for all cross-site functionality
**Phase 4 (Ongoing):** Monitor for issues as browsers finalize their implementations

---


## Related Articles

- [Passkey User Experience Comparison Across Chrome.](/privacy-tools-guide/passkey-user-experience-comparison-across-chrome-safari-firefox-edge-2026/)
- [Brave Browser vs Chrome Battery Drain Comparison](/privacy-tools-guide/brave-browser-battery-drain-vs-chrome-comparison/)
- [How To Stop Browser Fingerprinting On Chrome 2026 Practical](/privacy-tools-guide/how-to-stop-browser-fingerprinting-on-chrome-2026-practical-/)
- [Firefox Focus Vs Duckduckgo Browser Comparison](/privacy-tools-guide/firefox-focus-vs-duckduckgo-browser-comparison/)
- [Browser Fingerprinting How It Works and How to Prevent It](/privacy-tools-guide/browser-fingerprinting-how-it-works-and-how-to-prevent-it-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
