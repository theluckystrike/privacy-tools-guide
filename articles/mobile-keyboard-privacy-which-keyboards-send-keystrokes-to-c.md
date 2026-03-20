---
layout: default
title: "Mobile Keyboard Privacy: Which Keyboards Send Keystrokes."
description: "A technical comparison of mobile keyboard privacy policies. Learn which keyboards send keystrokes to cloud servers and how to protect your data."
date: 2026-03-16
author: theluckystrike
permalink: /mobile-keyboard-privacy-which-keyboards-send-keystrokes-to-c/
categories: [guides]
tags: [privacy-tools-guide, tools, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

# Mobile Keyboard Privacy: Which Keyboards Send Keystrokes to Cloud Servers

GBoard sends keystrokes to Google servers for predictions and emoji suggestions. SwiftKey, Gboard, and many keyboards transmit keystroke data to cloud services for personalization. Use locally-processing keyboards (Heliboard, OpenBoard) or analyze network traffic with Burp Suite to verify no data leaves your device before choosing.

## Understanding Keyboard Data Transmission

Mobile keyboards typically collect data for three purposes: improving prediction accuracy, enabling cloud-based features, and gathering analytics. The privacy implications vary significantly depending on whether processing happens locally or remotely.

When a keyboard sends keystrokes to cloud servers, your typing patterns, partial words, and potentially sensitive data traverse networks that you do not control. Even with encryption, metadata analysis can reveal information about your communication patterns.

### Local Processing vs Cloud Processing

Modern keyboards often use on-device machine learning for predictions. Apple's QuickType and Google's GBoard both use device-side processing for many features. However, certain capabilities—such as collaborative learning across devices or advanced language models—require server-side processing.

The distinction matters for threat models. If your primary concern is preventing your typing data from reaching third parties, local-only keyboards provide the strongest guarantees. If you accept some data collection but want transparency about what leaves your device, understanding each keyboard's architecture becomes critical.

## Major Keyboard Privacy Comparison

### GBoard (Google)

Google's GBoard sends keystroke data to Google's servers for several features. When you enable "Google Search" or use emoji predictions, your input may be transmitted. The keyboard stores usage statistics locally but syncs aggregated data to your Google account for personalization across devices.

You can examine GBoard's data handling through its settings:

```
Settings → Apps → GBoard → Permissions
Settings → Languages → Google Input Tools (disable for offline-only)
```

For developers, analyzing network traffic reveals that GBoard makes HTTPS requests to `clients3.google.com` and related domains during active typing sessions.

### SwiftKey (Microsoft)

Microsoft's SwiftKey historically transmitted typing data to improve its prediction algorithms. The application collects keystrokes, word sequences, and usage patterns, which Microsoft states are used to enhance prediction accuracy across users.

SwiftKey offers a "Privacy Dashboard" in its settings where you can view and delete collected data. However, cloud-based predictions remain enabled by default, meaning your typing data continues traveling to Microsoft's servers unless you explicitly disable these features.

### Apple Keyboard (iOS)

Apple's on-device keyboard processes nearly all predictions locally using the device's neural engine. iOS does not transmit keystroke data to external servers for keyboard predictions. This architecture provides stronger privacy guarantees compared to third-party alternatives.

Apple's keyboard does send usage statistics to Apple if you enable "Share Keyboard Usage Data" in settings. Developers can verify this by monitoring network traffic from the keyboard process, which shows minimal external communication during normal typing.

### Open Source Alternatives

For maximum privacy, several open-source keyboards process everything locally:

- **OpenBoard** (Android): Fully local processing, no network permissions required
- **AnySoftKeyboard** (Android): Open-source with optional cloud predictions you can disable
- **Hacker's Keyboard** (Android): Designed for developers with local dictionary only

These keyboards provide verifiable behavior since you can audit their source code or examine network traffic directly.

## Verifying Keyboard Network Behavior

For developers who want empirical data about keyboard behavior, several methods exist to observe network transmission.

### Android Network Inspection

Using Android's built-in debugging tools, you can monitor which servers your keyboard contacts:

```bash
# Enable USB debugging on your device
adb devices
adb shell pm list packages | grep keyboard

# Monitor network traffic from a specific package
adb shell am monitor -p com.google.android.inputmethod.latin
```

More detailed analysis requires setting up a proxy or using tools like Wireshark with a rooted device.

### iOS Network Analysis

iOS provides less visibility into individual app network connections. Developers can use Xcode's Network Instrument or third-party tools like Charles Proxy (requires certificate installation) to observe traffic:

1. Install Charles Proxy on your Mac
2. Configure your iOS device to use Charles as a proxy
3. Install Charles SSL certificate on your iOS device
4. Monitor traffic while typing in different keyboards

### Code-Level Verification

For a more thorough audit, examine the keyboard's source code if available:

```bash
# Check GBoard open-source components
git clone https://github.com/google/arabic-packages
# Review LatinIME library for network call patterns

# Examine AnySoftKeyboard source
git clone https://github.com/AnySoftKeyboard/AnySoftKeyboard
grep -r "HttpURLConnection\|OkHttp\|Retrofit" --include="*.java"
```

This approach reveals exactly what network calls the keyboard makes, though analyzing compiled binaries for closed-source keyboards requires reverse engineering.

## Practical Recommendations

Based on the privacy implications, here are actionable recommendations for different use cases.

### High-Security Environments

If you handle sensitive information—credentials, financial data, or confidential communications—use a keyboard with verified local-only processing:

1. **OpenBoard** for Android provides zero network permissions
2. **iOS default keyboard** with "Share Keyboard Usage Data" disabled
3. **Hardware keyboard** (Bluetooth) eliminates touchscreen keyboard data concerns entirely

### Balanced Privacy and Functionality

If you want predictive text while limiting data exposure:

1. Disable cloud-based predictions in any keyboard settings
2. Use local dictionary and personal learning only
3. Periodically clear keyboard data through app settings
4. Consider keyboards that allow you to self-host prediction models

### For Developers

When building applications that accept text input:

```kotlin
// Android: Suggest using IMM_FLAG_NO_EXTRACT_UI
// to prevent third-party keyboard access to full text fields

editText.imeOptions = EditorInfo.IME_FLAG_NO_EXTRACT_UI
editText.inputType = InputType.TYPE_TEXT_FLAG_NO_SUGGESTION
```

Implement password fields with `inputType="textPassword"` to ensure keyboards treat the input as sensitive data, which typically prevents prediction and transmission.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
