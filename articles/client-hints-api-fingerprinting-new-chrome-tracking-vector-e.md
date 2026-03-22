---
layout: default
title: "Client Hints API: The New Chrome Tracking Vector Explained"
description: "A technical deep-dive into the Client Hints API and how it enables fingerprinting. Learn what data Chrome exposes, how websites use it, and practical"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /client-hints-api-fingerprinting-new-chrome-tracking-vector-e/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api]
---

{% raw %}

The Client Hints API automatically sends your device model, operating system, CPU architecture, and browser version to every website via HTTP headers, enabling fingerprinting without JavaScript. Originally designed for content optimization, websites now use these headers to fingerprint and track users across sessions with high accuracy. Disable Client Hints through browser preferences (if available), use privacy extensions to block high-entropy hints, or use a privacy-focused browser like Firefox, which offers better Client Hints protection than Chrome.

## Table of Contents

- [What Are Client Hints?](#what-are-client-hints)
- [How Websites Use Client Hints for Fingerprinting](#how-websites-use-client-hints-for-fingerprinting)
- [Practical Fingerprinting Example](#practical-fingerprinting-example)
- [Browser Support and Availability](#browser-support-and-availability)
- [Defending Against Client Hints Tracking](#defending-against-client-hints-tracking)
- [What Developers Need to Know](#what-developers-need-to-know)
- [The Broader Privacy Sandbox Context](#the-broader-privacy-sandbox-context)
- [Fingerprinting Entropy Analysis](#fingerprinting-entropy-analysis)
- [Practical Fingerprinting Implementation](#practical-fingerprinting-implementation)
- [Cross-Site Tracking with Client Hints](#cross-site-tracking-with-client-hints)
- [Protection Mechanisms by Browser](#protection-mechanisms-by-browser)
- [Browser Extension Blocking Rules](#browser-extension-blocking-rules)
- [Advanced Fingerprinting via Hints Combination](#advanced-fingerprinting-via-hints-combination)
- [Privacy Sandbox and Future Tracking APIs](#privacy-sandbox-and-future-tracking-apis)
- [Recommendations by Threat Model](#recommendations-by-threat-model)

## What Are Client Hints?

Client Hints are HTTP headers that browsers send with each request, providing servers with information about the client device and browser configuration. Unlike traditional fingerprinting techniques that require JavaScript execution, Client Hints transmit this data automatically at the network level.

The API emerged from Google's broader Privacy Sandbox initiative, positioned as a privacy-respecting alternative to cross-site tracking. However, the implementation has raised substantial concerns among privacy researchers and developers.

### Available Client Hints Headers

Chrome exposes numerous hints that websites can request:

```
Sec-CH-UA: "Chrome"; v="120", "Not(A:Brand"; v="8"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: "Windows"
Sec-CH-UA-Platform-Version: "15.0.0"
Sec-CH-UA-Arch: "x86"
Sec-CH-UA-Bitness: "64"
Sec-CH-UA-Model: "Galaxy S21"
Sec-CH-UA-WOW64: ?0
Sec-CH-Prefers-Color-Scheme: "dark"
Sec-CH-Prefers-Reduced-Motion: "no-preference"
Viewport-Width: 1920
Device-Memory: 8
Downlink: 10.85
ECT: 4g
RTT: 50
```

Each header reveals specific information about your device and preferences. The `Sec-CH-UA-*` headers expose detailed browser and platform information, while `Device-Memory` reveals available RAM, and network-related hints like `Downlink` and `ECT` characterize your connection quality.

## How Websites Use Client Hints for Fingerprinting

When you visit a site that implements Client Hints fingerprinting, the server responds with an `Accept-CH` header requesting specific hints. Chrome then includes these headers in subsequent requests to that origin.

Here's a practical example of how a site might implement hint-based fingerprinting:

```javascript
// Server sends this response header to request Client Hints
Accept-CH: Sec-CH-UA, Sec-CH-UA-Mobile, Sec-CH-UA-Platform,
           Sec-CH-UA-Model, Device-Memory, Downlink

// JavaScript can also access hints via the Permissions API
async function getClientHints() {
  if (!window.clientHintsInfrastructure) return;

  const hints = await document.clientHintsInfrastructure.getHintsDeprecation();
  console.log('Available hints:', hints);
}
```

The fingerprinting potential comes from combining multiple hints. While individual hints might seem innocuous, the combination creates a highly unique identifier:

| Hint | Information Revealed |
|------|---------------------|
| Sec-CH-UA | Browser brand and version |
| Sec-CH-UA-Model | Device model |
| Sec-CH-UA-Platform | Operating system |
| Device-Memory | RAM amount |
| Sec-CH-UA-Arch | CPU architecture |

A user with a specific Chrome version, on a particular device model, running Windows 11 with 16GB RAM becomes significantly more identifiable than someone using generic values.

## Practical Fingerprinting Example

Consider this scenario: a website wants to build a fingerprint:

```python
# Server-side example (Python/Flask)
@app.route('/track')
def track_fingerprint():
    user_hints = {
        'ua': request.headers.get('Sec-CH-UA'),
        'mobile': request.headers.get('Sec-CH-UA-Mobile'),
        'platform': request.headers.get('Sec-CH-UA-Platform'),
        'model': request.headers.get('Sec-CH-UA-Model'),
        'memory': request.headers.get('Device-Memory'),
        'arch': request.headers.get('Sec-CH-UA-Arch'),
    }

    # Create fingerprint hash from combined hints
    fingerprint = hashlib.sha256(
        '|'.join(str(v) for v in user_hints.values() if v)
    ).hexdigest()

    # Store or correlate with existing user data
    return jsonify({'fingerprint': fingerprint})
```

This approach works even when users clear cookies, employ VPN services, or use private browsing modes. The fingerprint derives entirely from headers that Chrome sends automatically.

## Browser Support and Availability

Client Hints have expanded significantly across Chromium-based browsers:

- **Chrome/Edge**: Full support, actively deploying new hints
- **Opera**: Inherits Chromium implementation
- **Brave**: Blocks many hints by default
- **Firefox/Safari**: Limited or no support

This uneven support creates additional fingerprinting opportunities. Sites can detect which hints a browser provides (or doesn't provide), using the presence or absence of specific hints as another distinguishing factor.

## Defending Against Client Hints Tracking

Several strategies can reduce Client Hints fingerprinting:

### Browser Configuration

Firefox provides the most protection by not implementing Client Hints. For Chrome users, the built-in privacy settings offer limited control:

1. Navigate to `chrome://settings/cookies` and disable "Send Do Not Track" (ironically, this reduces your fingerprint surface)
2. Use browser extensions that modify request headers

### Extension-Based Blocking

Extensions like uBlock Origin can block or modify Client Hints headers:

```
! Block Device-Memory header
||example.com^$header=Device-Memory

! Block all Client Hints infrastructure
||example.com^$header=Sec-CH-UA
||example.com^$header=Sec-CH-UA-Mobile
||example.com^$header=Sec-CH-UA-Platform
```

### Network-Level Solutions

For advanced users, proxy services and privacy-focused DNS providers can filter or normalize Client Hints before they reach their destination.

### The Tor Browser Approach

Tor Browser provides the most protection by standardizing Client Hints across all users. Every Tor Browser instance reports identical hint values, eliminating the ability to distinguish between users based on these headers.

## What Developers Need to Know

If you're building web applications, consider the privacy implications before implementing Client Hints:

1. **Minimal necessary hints**: Only request hints actually needed for content optimization
2. **Graceful degradation**: Design systems to work when hints aren't available
3. **User consent**: Consider whether you genuinely need device fingerprinting versus less invasive analytics

## The Broader Privacy Sandbox Context

Client Hints exist within Google's larger Privacy Sandbox framework, which includes APIs like Topics, Attribution Reporting, and FLEDGE. While marketed as privacy improvements, these APIs create new tracking mechanisms that operate without traditional cookies.

The fundamental tension: Chrome positions Client Hints as a privacy-respecting alternative to fingerprinting, while the API itself enables fingerprinting at scale.

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

- [Chrome Privacy Sandbox Explained What It Means For Tracking](/privacy-tools-guide/chrome-privacy-sandbox-explained-what-it-means-for-tracking-/)
- [Attribution Reporting API How Chrome Replaced Cookies](/privacy-tools-guide/attribution-reporting-api-how-chrome-replaced-cookies-for-ad/)
- [Topics API Chrome Replacement For Cookies How It Tracks](/privacy-tools-guide/topics-api-chrome-replacement-for-cookies-how-it-tracks-you/)
- [Device Memory API Fingerprinting How Ram Amount Narrows](/privacy-tools-guide/device-memory-api-fingerprinting-how-ram-amount-narrows-iden/)
- [How Browser Fingerprinting Works Explained](/privacy-tools-guide/how-browser-fingerprinting-works-explained/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
## Fingerprinting Entropy Analysis

To understand the tracking risk, calculate the entropy (uniqueness) of your fingerprint:

```python
import hashlib
import math

def calculate_fingerprint_entropy(hints: dict) -> float:
    """Calculate entropy (bits of information) in your Client Hints fingerprint"""

    # Estimate possible values for each hint
    hint_distributions = {
        'Sec-CH-UA': 50,                    # ~50 distinct browser/version combos
        'Sec-CH-UA-Mobile': 2,              # binary: mobile or not
        'Sec-CH-UA-Platform': 5,            # Windows, Mac, Linux, Android, iOS
        'Sec-CH-UA-Platform-Version': 100,  # many OS versions
        'Sec-CH-UA-Arch': 3,                # x86, ARM, other
        'Sec-CH-UA-Bitness': 2,             # 32-bit or 64-bit
        'Sec-CH-UA-Model': 1000,            # thousands of device models
        'Device-Memory': 8,                 # 2GB, 4GB, 8GB, 16GB, etc
    }

    # Calculate total entropy
    total_entropy = 0
    for hint, possible_values in hint_distributions.items():
        if hints.get(hint):
            # Entropy = log2(possible_values)
            entropy = math.log2(possible_values)
            total_entropy += entropy
            print(f"{hint}: {entropy:.1f} bits")

    print(f"\nTotal fingerprint entropy: {total_entropy:.1f} bits")
    print(f"Uniqueness: 1 in {2**total_entropy:,.0f} browsers")

    return total_entropy

# Example
my_hints = {
    'Sec-CH-UA': '"Chrome"; v="120", "Not(A:Brand"; v="8"',
    'Sec-CH-UA-Platform': 'Linux',
    'Sec-CH-UA-Arch': 'x86',
    'Device-Memory': 16,
    'Sec-CH-UA-Model': 'Custom Desktop'
}

entropy = calculate_fingerprint_entropy(my_hints)
```

With 25-30 bits of entropy, your fingerprint uniquely identifies you among thousands of users. Modern browsers combined with Client Hints easily exceed this threshold.

## Practical Fingerprinting Implementation

A real website using Client Hints for tracking would implement:

```javascript
// Server-side tracking implementation (Python/Node.js)
class ClientFingerprintTracker {
    constructor(database) {
        this.db = database;
    }

    extract_hints_from_headers(request) {
        // Extract from HTTP headers
        return {
            ua: request.headers.get('Sec-CH-UA'),
            mobile: request.headers.get('Sec-CH-UA-Mobile'),
            platform: request.headers.get('Sec-CH-UA-Platform'),
            model: request.headers.get('Sec-CH-UA-Model'),
            memory: request.headers.get('Device-Memory'),
            arch: request.headers.get('Sec-CH-UA-Arch'),
            color_scheme: request.headers.get('Sec-CH-Prefers-Color-Scheme'),
            viewport_width: request.headers.get('Viewport-Width'),
            network_type: request.headers.get('ECT')
        };
    }

    create_fingerprint(hints) {
        // Combine hints into a unique identifier
        import hashlib
        combined = '|'.join(str(v) for v in hints.values())
        fingerprint_hash = hashlib.sha256(combined.encode()).hexdigest()
        return fingerprint_hash

    def track_user(self, request, user_id=None):
        hints = self.extract_hints_from_headers(request)
        fingerprint = self.create_fingerprint(hints)

        # Store or update fingerprint record
        self.db.upsert('fingerprints', {
            'fingerprint_hash': fingerprint,
            'user_id': user_id,
            'hints': hints,
            'last_seen': datetime.now(),
            'visit_count': self.db.get_visit_count(fingerprint) + 1
        })

        # Correlate with other tracking data
        # (cookies, pixel tracking, analytics tags)
        return fingerprint
```

This creates a persistent identifier that survives:
- Cookie deletion
- Private browsing sessions
- Browser cache clearing
- VPN changes (but NOT IP address changes)

## Cross-Site Tracking with Client Hints

The dangerous part: Hints are sent to EVERY website you visit:

```
User visits site1.com
→ site1.com receives Client Hints header
→ site1.com records fingerprint

User visits site2.com (different website)
→ site2.com receives SAME Client Hints header
→ site2.com records fingerprint

site1.com + site2.com use data brokers to share fingerprints
→ User is now tracked across multiple websites
→ No cookies needed; no consent required
```

This is fundamentally different from cookie tracking because:
1. Users expect cookies can be deleted
2. Hints are "just device information" users don't think about
3. Technical prevention requires browser-level changes, not user action

## Protection Mechanisms by Browser

### Firefox (Best Protection)

Firefox doesn't implement Client Hints, providing strongest protection:

```javascript
// Firefox ignores Accept-CH headers
// Fingerprinting attempts return generic values
navigator.userAgent
// "Mozilla/5.0 (X11; Linux x86_64; rv:121.0) Gecko/20100101 Firefox/121.0"
// No detailed hints available
```

**Recommendation:** Use Firefox for high-privacy browsing.

### Brave (Good Protection)

Brave blocks high-entropy hints by default:

```
Blocked by default:
  - Sec-CH-UA-Model (device model)
  - Sec-CH-UA-Full-Version-List
  - Device-Memory
  - Sec-CH-UA-Platform-Version

Allowed (low-entropy):
  - Sec-CH-UA (browser, already in User-Agent)
  - Sec-CH-UA-Mobile (binary value)
  - Sec-CH-UA-Platform (OS, already in User-Agent)
```

**Recommendation:** Use Brave for Chrome-based browsing with privacy.

### Chrome (Limited Protection)

Chrome's privacy controls are minimal:

```
chrome://settings/privacy
→ "Send Do Not Track" - Adds header but doesn't block hints
→ No Client Hints specific controls

Solution: Use extensions to block or modify hints
```

**Recommendation:** If using Chrome, install uBlock Origin with custom rules.

## Browser Extension Blocking Rules

### uBlock Origin Rules

```
! Block all Client Hints
||example.com^$header=Sec-CH-UA
||example.com^$header=Sec-CH-UA-Mobile
||example.com^$header=Sec-CH-UA-Platform
||example.com^$header=Sec-CH-UA-Platform-Version
||example.com^$header=Sec-CH-UA-Arch
||example.com^$header=Sec-CH-UA-Bitness
||example.com^$header=Sec-CH-UA-Model
||example.com^$header=Device-Memory
||example.com^$header=Downlink
||example.com^$header=ECT
||example.com^$header=RTT
```

Note: uBlock Origin's header-blocking feature has limitations. Full blocking requires more advanced solutions.

### Header Modification with LibreWolf

LibreWolf (Firefox hardened fork) removes Client Hints entirely:

```
about:config settings:
  network.http.http2.enabled = true
  network.http.http3.enabled = false
  # Client Hints already disabled in Firefox
```

## Advanced Fingerprinting via Hints Combination

Sophisticated trackers combine Client Hints with other signals:

```python
def create_advanced_fingerprint(request):
    """Combine Client Hints with other signals"""

    hints_fingerprint = hashlib.sha256(
        f"{request.headers.get('Sec-CH-UA')}" +
        f"{request.headers.get('Device-Memory')}" +
        f"{request.headers.get('Sec-CH-UA-Model')}"
    ).hexdigest()

    # Layer 1: Client Hints
    fp_layer1 = hints_fingerprint

    # Layer 2: HTTP Accept headers (font types, image formats)
    fp_layer2 = hashlib.sha256(
        request.headers.get('Accept')
    ).hexdigest()

    # Layer 3: TLS fingerprint (cipher suite preferences)
    fp_layer3 = request.tls_fingerprint

    # Layer 4: IP geolocation
    fp_layer4 = request.client_ip

    # Combine all layers
    complete_fingerprint = hashlib.sha256(
        f"{fp_layer1}{fp_layer2}{fp_layer3}{fp_layer4}"
    ).hexdigest()

    return complete_fingerprint
```

This multi-layer approach makes fingerprints persistent even if one signal changes.

## Privacy Sandbox and Future Tracking APIs

Client Hints are part of Google's larger Privacy Sandbox initiative, which includes:

1. **FLEDGE** (Federated Learning of Cohorts): Group users into interest cohorts
2. **Topics API**: Website shares inferred topics about user interests
3. **Attribution Reporting**: Tracks conversions without cross-site cookies
4. **Aggregate Reporting**: Anonymous aggregate statistics

All these APIs face similar concerns: they provide tracking capabilities without user consent or ability to opt-out technically.

## Recommendations by Threat Model

| Threat Model | Browser | Extensions |
|-------------|---------|-----------|
| Casual user | Chrome | uBlock Origin |
| Privacy-conscious | Brave | Default protections |
| Privacy-researcher | Firefox + LibreWolf | Minimal extensions |
| Journalist/Activist | Tor Browser | Already hardened |

For maximum protection, Tor Browser standardizes all Client Hints to identical values, eliminating fingerprinting differences.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
