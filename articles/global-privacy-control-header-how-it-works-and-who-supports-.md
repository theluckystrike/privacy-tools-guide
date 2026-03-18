---
layout: default
title: "Global Privacy Control Header: How It Works and Who."
description: "A technical guide to the GPC header for developers. Learn how this privacy signal works, which browsers and companies support it, and how to implement it."
date: 2026-03-16
author: theluckystrike
permalink: /global-privacy-control-header-how-it-works-and-who-supports-/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

The Global Privacy Control (GPC) header represents a significant advancement in user privacy signaling on the web. Unlike its predecessor Do Not Track (DNT), GPC carries legal weight under several privacy regulations, making it a more practical tool for privacy-conscious users and developers alike.

## What Is Global Privacy Control?

Global Privacy Control is an HTTP header that browsers send to indicate a user's preference to opt out of data selling and targeted advertising. Unlike DNT, which was purely advisory, GPC has regulatory backing:

- **California Consumer Privacy Act (CCPA)**: California's attorney general stated that GPC signals satisfy the "opt-out of sale" requirement
- **Virginia Consumer Data Protection Act (VCDPA)**: Recognizes GPC as a valid opt-out signal
- **Colorado Privacy Act**: Also acknowledges GPC as a legitimate privacy preference

This legal recognition distinguishes GNC from earlier privacy headers and makes compliance more straightforward for businesses.

## How the GPC Header Works

When a user enables privacy protections in their browser, every HTTP request includes the GPC header. The header value is straightforward:

```
Sec-GPC: 1
```

The `Sec-` prefix indicates this is a fetch metadata header, providing additional security context about the request.

### Detecting GPC in JavaScript

You can check for GPC support using the `navigator.globalPrivacyControl` property:

```javascript
if (navigator.globalPrivacyControl === true) {
  console.log('User has enabled Global Privacy Control');
  // Adjust tracking/personalization accordingly
}
```

This property returns `true` when the user has opted out, `false` when they haven't, and `undefined` when the browser doesn't support GPC.

### Server-Side Detection

On the server, you can detect the GPC header in your request processing:

**Node.js/Express:**

```javascript
app.get('/api/content', (req, res) => {
  const gpcValue = req.headers['sec-gpc'];
  
  if (gpcValue === '1') {
    // User has opted out of data sale
    // Disable analytics, personalization, and third-party sharing
    disableTracking(req.userId);
    disablePersonalization(req.sessionId);
  }
  
  res.json({ /* content */ });
});
```

**Python/Flask:**

```python
@app.route('/api/content')
def get_content():
    gpc_header = request.headers.get('Sec-GPC')
    
    if gpc_header == '1':
        # Respect user's privacy preference
        disable_tracking()
        disable_personalization()
    
    return jsonify({ /* content */ })
```

**PHP:**

```php
<?php
$gpc_value = $_SERVER['HTTP_SEC_GPC'] ?? null;

if ($gpc_value === '1') {
    // Respect privacy preference
    disable_analytics();
    disable_ad_tracking();
}
?>
```

### Respecting GPC in Cookie Consent

If you manage cookies through a consent platform, GPC should override consent preferences:

```javascript
function shouldBlockTracking() {
  // GPC takes precedence over cookie consent
  if (navigator.globalPrivacyControl === true) {
    return true;
  }
  
  // Fall back to cookie consent check
  return !hasCookieConsent();
}
```

## Browser Support for Global Privacy Control

GPC support varies across browsers. Here's the current landscape:

### Desktop Browsers

- **Brave**: Full support, enabled by default in all privacy settings
- **Firefox**: Native support, user must enable in settings
- **Edge**: Supports GPC but not enabled by default
- **Chrome**: No native GPC support as of early 2026
- **Safari**: Partial support through Intelligent Tracking Prevention

### Mobile Browsers

- **Brave (iOS/Android)**: Full support
- **Firefox (iOS/Android)**: Supported
- **Safari (iOS)**: Integrates with iOS privacy features
- **Chrome (Android)**: Limited support

### Browser Implementation Details

Users typically find GPC settings in:

- **Brave**: Settings → Privacy and security → "Send Global Privacy Control signal"
- **Firefox**: Settings → Privacy & Security → "Tell websites not to sell or share my data"

## Who Supports GPC? Companies and Platforms

The list of companies honoring GPC signals has grown significantly:

### Major Platforms

- **Google**: Honors GPC for California users through AdSettings
- **Meta**: Respects GPC for covered jurisdictions
- **Amazon**: Implements GPC for advertising personalization
- **Microsoft**: Applies GPC across its advertising ecosystem

### Ad Networks and Trackers

- **Google Ads**: Processes GPC signals in covered states
- **Meta Ads**: Recognizes GPC for targeted advertising
- **Trade Desk**: Supports GPC as a universal opt-out signal
- **LiveRamp**: Honored GPC before retiring third-party data

### Tools and Frameworks

Most modern consent management platforms (CMPs) respect GPC:

- **OneTrust**: Automatically detects and honors GPC
- **Cookiebot**: Processes GPC alongside consent signals
- **TrustArc**: Recognizes GPC as a valid opt-out mechanism

## Implementing GPC on Your Website

If you run a website, here's how to properly handle GPC:

### Step 1: Detect the Signal

Add server-side logic to check for the `Sec-GPC` header on incoming requests.

### Step 2: Disable Tracking

When GPC is detected, ensure you:

- Don't set advertising cookies
- Disable analytics personalization
- Don't share data with third parties
- Don't sell user data

### Step 3: Communicate Compliance

Add a notice in your privacy policy acknowledging GPC support:

> "We respect the Global Privacy Control (GPC) signal. When detected, we automatically disable all tracking, personalization, and data sharing that would constitute a sale under applicable privacy laws."

## Limitations of GPC

GPC isn't a complete privacy solution. Be aware of these constraints:

1. **Geographic limitations**: Legal requirements apply only in specific jurisdictions
2. **First-party tracking**: GPC doesn't block all tracking—just cross-site sharing
3. **Implementation gaps**: Some companies still ignore GPC despite legal requirements
4. **No fingerprinting protection**: GPC doesn't prevent browser fingerprinting

## Testing GPC Implementation

Verify your GPC handling works correctly:

```bash
# Test with curl
curl -H "Sec-GPC: 1" https://yourwebsite.com

# Check response headers for proper processing
# Verify no tracking cookies are set
```

Browser developer tools also show the GPC header in the Network tab when making requests.

## Conclusion

The Global Privacy Control header provides a standardized, legally recognized way for users to opt out of data selling. For developers, implementing GPC support is straightforward and often satisfies regulatory requirements. While not a complete privacy solution, GPC represents meaningful progress in giving users control over their digital footprint.

Browser support continues to expand, and regulatory pressure will likely increase adoption among tracking companies. Getting GPC right today prepares your infrastructure for evolving privacy expectations tomorrow.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
