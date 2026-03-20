---
layout: default
title: "Hsts Supercookie Tracking How Https Settings Create Persiste"
description: "Learn how HSTS supercookies work and how HTTPS settings can be exploited to create persistent tracking identifiers across websites."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /hsts-supercookie-tracking-how-https-settings-create-persiste/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

HSTS supercookies exploit the browser's HSTS preload list (which persists locally) to encode tracking identifiers: a site sends HSTS headers that instruct the browser to remember dozens of related domains, and the pattern of which domains are recorded in HSTS creates a unique fingerprint. This persists across private browsing, cookie deletion, and even browser resets. Mitigation: regularly clear your HSTS cache (though most modern browsers reset it in private mode), use privacy extensions that block HSTS state storage, or switch to Tor Browser which isolates HSTS per-session.

## How HSTS Supercookies Work

HSTS works through a header that servers send to browsers:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

When a browser receives this header, it records the domain in its HSTS preload list for the specified duration. The critical privacy implication: browsers store this information persistently, and the presence or absence of HSTS for a domain can encode binary information.

### Encoding Data as HSTS State

Attackers can exploit this by controlling multiple subdomains, each sending different HSTS policies. A tracker sets HSTS on some subdomains while leaving others unset. When the user visits an identifier page, the tracker queries these subdomains and observes whether the browser attempts HTTPS connections (indicating HSTS is active) or falls back to HTTP.

This creates a binary encoding system:

- Subdomain with HSTS enabled = 1
- Subdomain without HSTS = 0

With enough subdomains, you can encode unique identifiers. A 32-subdomain setup provides over 4 billion unique combinations—enough to track users indefinitely without traditional cookies.

## Practical Example

Consider a tracker controlling these subdomains:

```
track-0.example.com
track-1.example.com
track-2.example.com
...
track-31.example.com
```

The tracker encodes a 32-bit user ID by setting HSTS headers selectively:

```javascript
// Server-side code (Node.js) demonstrating the encoding
function setHSTSForUserId(userId) {
  const subdomains = [];
  for (let i = 0; i < 32; i++) {
    const bit = (userId >> i) & 1;
    if (bit === 1) {
      subdomains.push(`Strict-Transport-Security: max-age=31536000; includeSubDomains`);
    }
  }
  return subdomains;
}

// On first visit, set HSTS based on user ID
app.get('/init', (req, res) => {
  const userId = generateUniqueId(); // 32-bit identifier
  res.set('Strict-Transport-Security', `max-age=31536000; includeSubDomains`);
  // Redirect through each subdomain to set HSTS state
  res.redirect(`/setup?id=${userId}`);
});
```

The browser then remembers these HSTS settings for up to one year.

## Detecting HSTS Supercookies

Power users can inspect HSTS state in their browsers:

**Chrome**: Navigate to `chrome://net-internals/#hsts`

**Firefox**: Type `about:networking` in the address bar and check the HSTS section

**Command-line verification**:

```bash
# Query HSTS state using Chrome's net-internals
echo "example.com" | grep -q "HSTS" && echo "HSTS enabled" || echo "No HSTS"
```

## Mitigation Strategies

Several approaches help prevent HSTS supercookie tracking:

### 1. Clear HSTS State

Regularly clear HSTS data through browser settings. In Chrome, you can delete HSTS domains via `chrome://net-internals/#hsts` by typing a domain and checking its status, then using the "Delete domain security policies" option.

### 2. Use Firefox's Enhanced Tracking Protection

Firefox blocks known supercookie tracking by default. Enable strict tracking protection in settings.

### 3. Browser Extensions

Extensions like "Forget Me Not" or "Privacy Badger" attempt to neutralize supercookie vectors.

### 4. Network-Level Blocking

Configure Pi-hole or similar DNS-level blockers to prevent connections to known tracking subdomains.

## The Broader Implications

HSTS supercookies represent a fundamental tension in web security. The same mechanism that protects users from protocol downgrade attacks enables fingerprinting that bypasses traditional privacy controls. Unlike cookies, HSTS state persists through:

- Private/incognito browsing sessions
- Cookie deletion
- Browser cache clearing
- Browser resets (until HSTS preload lists are cleared)

This makes HSTS supercookies particularly difficult to combat. Security researchers continue to propose solutions, including randomizing HSTS behavior or requiring explicit user consent before HSTS policies are applied.

## Testing Your Browser

You can verify your browser's vulnerability using online testing tools:

```bash
# Using curl to check HSTS headers (manual verification)
curl -I https://example.com 2>/dev/null | grep -i "Strict-Transport-Security"
```

If you see the HSTS header, your browser will remember this domain for the specified duration.

## HSTS Preload Lists and Long-Term Persistence

The HSTS preload list takes supercookie persistence to another level. Major browsers ship with a compiled list of domains that should always use HTTPS—this list is hardcoded into browser binaries and cannot be easily cleared by users. While the preload list itself isn't designed for tracking, it demonstrates how deeply embedded HTTP redirection can become.

For developers, understanding preload implications matters:

```javascript
// Check if a domain is on Chrome's HSTS preload list
// This cannot be cleared by users
const preloadDomains = [
  'google.com', 'facebook.com', 'twitter.com', 
  // ... thousands of other domains
];

function isPreloaded(domain) {
  return preloadDomains.includes(domain.toLowerCase());
}
```

Preloaded domains automatically enforce HTTPS for one year minimum, creating persistent connection behavior that cannot be overridden without browser updates.

## Developer Considerations

Web developers should understand both the security benefits and privacy concerns of HSTS:

- **Security benefit**: Prevents man-in-the-middle attacks and protocol downgrades
- **Privacy concern**: Creates a persistent state that trackers can exploit
- **Best practice**: Use HSTS, but be aware of how it interacts with tracking prevention tools

When implementing HSTS in your applications, consider using shorter max-age values during development:

```nginx
# Nginx configuration for HSTS
add_header Strict-Transport-Security "max-age=3600; includeSubDomains" always;
# Development: short max-age
# Production: max-age=31536000; includeSubDomains; preload
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
