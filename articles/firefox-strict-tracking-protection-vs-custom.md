---
layout: default
title: "Firefox Strict Tracking Protection vs Custom: A."
description: "Compare Firefox's built-in Strict tracking protection with custom about:config configurations. Learn practical settings for developers and power users."
date: 2026-03-15
author: theluckystrike
permalink: /firefox-strict-tracking-protection-vs-custom/
categories: [guides]
reviewed: true
score: 8
---

{% raw %}

Firefox offers multiple approaches to blocking trackers, from the simple Strict preset to granular about:config tuning. For developers and power users, understanding the tradeoffs between these approaches helps you balance privacy, compatibility, and performance.

## Understanding Firefox's Tracking Protection

Firefox's Enhanced Tracking Protection (ETP) blocks known trackers across three levels: Standard, Strict, and Custom. The Strict setting provides one-click protection, while Custom mode lets you pick individual blocking categories.

The core technology uses Disconnect.me's tracker lists, categorizing trackers into social media trackers, cross-site tracking cookies, fingerprinters, and cryptominers. Firefox maintains these lists and updates them regularly through Firefox Monitor.

## The Strict Setting

The Strict preset enables all tracking protections with a single click. Here's what it blocks:

- Cross-site tracking cookies (third-party cookies)
- Social media trackers embedded in pages
- Fingerprinting scripts that create persistent device identifiers
- Cryptominers that hijack your CPU

To enable Strict mode, navigate to **Settings > Privacy & Security** and select "Strict" under Enhanced Tracking Protection.

The Strict setting is the fastest path to improved privacy. However, it can break some websites. News sites, e-commerce platforms, and web applications that rely on third-party analytics or advertising scripts may malfunction. This is where Custom configuration becomes valuable.

## Building a Custom Configuration

Custom mode lets you selectively enable protections. Access it by choosing "Custom" in the same settings location. You'll see four toggleable categories:

- **Cookies**: Cross-site tracking cookies only, or all third-party cookies
- **Tracking content**: Block in all windows or tabs
- **Fingerprinters**: Standard or Strict (strict blocks more aggressively)
- **Cryptominers**: Block or allow

For most developers and power users, a Custom configuration strikes the right balance. Here's a practical starting point:

### Recommended Custom Settings

```about:config
# Enable cross-site tracking cookie blocking
privacy.trackingprotection.cookies.enabled = true

# Block tracking content in all windows (not just private)
privacy.trackingprotection.enabled = true

# Enable fingerprinting protection
privacy.resistFingerprinting = true

# Block cryptominers
privacy.trackingprotection.cryptomining.enabled = true
```

Navigate to `about:config` in your Firefox address bar to modify these values. Each setting has a boolean or integer value you can toggle.

## Advanced about:config Settings

Beyond the basic toggles, Firefox exposes dozens of advanced preferences for fine-grained control. Here are the most useful for power users:

### Enhanced Tracking Protection Overrides

```about:config
# Allow specific domains to bypass ETP
privacy.trackingprotection.annotate_channels = true
```

When enabled, Firefox adds a shield icon to the address bar showing blocked trackers. Clicking it reveals which domains were blocked, useful for debugging.

### Cookie Management

```about:config
# Keep cookies until Firefox closes (session cookies)
network.cookie.cookieBehavior = 1

# Block third-party cookies entirely
network.cookie.thirdparty.sessionOnly = true

# Disable cookie jar partitioning (may break some sites)
privacy.partition.always_partition_third_party = false
```

Cookie partitioning isolates third-party cookies by the first-party site you visited. This prevents cross-site tracking but can break legitimate uses like embedded widgets.

### Fingerprinting Resistance

```about:config
# Reduce fingerprinting surface
privacy.resistFingerprinting = true

# Spoof timezone to generic value
privacy.resistFingerprinting.timezoneOverride = "UTC"

# Reduce viewport variations reported to websites
privacy.resistFingerprinting.widthOverride = 1000
```

The `privacy.resistFingerprinting` setting is particularly powerful. It normalizes many browser characteristics that fingerprinting scripts use to create unique device identifiers.

### Content Blocking Extensions

```about:config
# Enable uBlock Origin compatibility mode
extensions.webextensions.enabled = true

# Allow content blocking extensions in private windows
privacy.webextensions.content允許.list = ["*"]
```

Firefox's content blocking works alongside extensions like uBlock Origin. For developers testing both, understanding how they interact matters.

## Testing Your Configuration

After configuring settings, verify they're working. Firefox provides built-in tools:

1. **Protection Dashboard**: Visit `about:protections` to see blocking statistics
2. **Inspector Tool**: Open Developer Tools (F12), go to the Storage tab, and examine cookies
3. **Network Monitor**: Check for blocked requests in the Network tab

Here's a simple JavaScript snippet to check if fingerprinting protection is active:

```javascript
// Check if fingerprinting resistance is enabled
const screenProps = [
  screen.colorDepth,
  screen.pixelDepth,
  screen.width,
  screen.height
];

// If these differ from expected, protection is working
console.log('Screen properties:', screenProps);
console.log('User agent:', navigator.userAgent);
```

Compare values between protected and unprotected Firefox instances to confirm protection is active.

## Common Compatibility Issues

Custom configurations sometimes cause website issues. Here are typical problems and solutions:

**Broken login sessions**: Third-party cookies blocked. Add the site to exceptions or temporarily allow third-party cookies.

**Embedded content fails**: Videos or maps don't load. Check which tracker category is blocking the embedded resource.

**WebSocket connections fail**: Some real-time applications use tracking domains. Examine WebSocket URLs in the Network tab.

**Development localhost issues**: Local development may trigger tracking protection. Use `http://localhost` with appropriate exceptions during development.

## Performance Considerations

Blocking trackers actually improves page load times. Third-party tracker scripts often add significant latency through additional DNS lookups, connections, and JavaScript execution. Strict protection typically reduces page weight by 20-50% on tracker-heavy sites.

However, aggressive fingerprinting protection can cause slight rendering delays as Firefox normalizes browser characteristics. The tradeoff is usually worthwhile for privacy-conscious users.

## When to Use Strict vs Custom

Choose Strict mode when you want maximum protection with minimal configuration effort and can tolerate occasional website breakage.

Choose Custom mode when you need to maintain specific site functionality while still blocking trackers. Developers often need Custom mode to test their own sites properly.

The optimal configuration depends on your threat model and workflow. Most power users benefit from a Custom setup that enables cross-site cookie blocking and fingerprinting protection while allowing exceptions for development and critical web applications.


## Related Reading

- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [VPN Warrant Canary: What It Means and Why It Matters](/privacy-tools-guide/vpn-warrant-canary-what-it-means/)
- [Best VPN for Linux Desktop: A Developer Guide](/privacy-tools-guide/best-vpn-for-linux-desktop/)

Built by the luckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
Built by theluckystrike — More at [zovo.one](https://zovo.one)
