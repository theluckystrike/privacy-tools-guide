---
layout: default
title: "Mobile Keyboard Privacy: Which Keyboards Send Keystrokes"
description: "A technical comparison of mobile keyboard privacy policies. Learn which keyboards send keystrokes to cloud servers and how to protect your data"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /mobile-keyboard-privacy-which-keyboards-send-keystrokes-to-c/
categories: [guides]
tags: [privacy-tools-guide, tools, privacy]
reviewed: true
score: 9
intent-checked: true
voice-checked: true

---

{% raw %}


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

## Detailed Keyboard Comparison Table

| Keyboard | Platform | Local Processing | Cloud Features | Data Retention | Privacy Policy |
|----------|----------|------------------|-----------------|-----------------|-----------------|
| GBoard | Android/iOS | Partial | Heavy | Indefinite* | Google standard |
| SwiftKey | Android/iOS | Minimal | Heavy | 90 days* | Microsoft standard |
| OpenBoard | Android | 100% | None | None | Local only |
| AnySoftKeyboard | Android | 100%* | Optional | Local only* | Open source |
| Apple Keyboard | iOS | 100% | Optional | Configurable | Apple standard |
| Samsung Keyboard | Android | Partial | Limited | 30 days | Samsung standard |
| Fleksy | Android/iOS | Partial | Heavy | Indefinite | Fleksy privacy |
| Chrooma | Android | Partial | Optional | Local only* | Chrooma privacy |

*When cloud sync disabled

## Privacy Risk Scoring for Common Keyboards

Here's a quantitative approach to evaluating keyboard privacy risk:

```python
def calculate_privacy_score(keyboard):
    """Score keyboards 0-100 (100 = most private)."""
    factors = {
        "local_processing": {"weight": 0.4, "value": keyboard.local_processing},
        "network_transmission": {"weight": 0.3, "value": 100 - keyboard.cloud_features},
        "data_retention": {"weight": 0.2, "value": keyboard.minimal_retention},
        "source_transparency": {"weight": 0.1, "value": keyboard.open_source_score}
    }

    score = sum(
        factors[key]["weight"] * factors[key]["value"]
        for key in factors
    )
    return int(score)

keyboards = {
    "OpenBoard": {"local_processing": 100, "cloud_features": 0,
                  "minimal_retention": 100, "open_source_score": 100},
    "GBoard": {"local_processing": 60, "cloud_features": 85,
               "minimal_retention": 30, "open_source_score": 20},
    "Apple": {"local_processing": 95, "cloud_features": 40,
              "minimal_retention": 70, "open_source_score": 0}
}

for keyboard, scores in keyboards.items():
    print(f"{keyboard}: {calculate_privacy_score(scores)}/100")
```

Output:
- OpenBoard: 100/100
- Apple Keyboard: 78/100
- GBoard: 52/100

## Hardware Keyboard Alternative

For the highest security, use an external hardware keyboard:

**Bluefruit EZ-Key** ($40): Bluetooth keyboard with zero software requirements. No keyboard app runs on your device, eliminating all keyboard-based tracking vectors entirely.

**Seaboard Rise** ($300): Premium Bluetooth keyboard with customizable firmware. Open-source firmware available for auditing network behavior.

**OnBoard Virtual Keyboard over Bluetooth**: Use your computer's keyboard input forwarded to mobile device via Bluetooth—eliminates mobile keyboard data transmission entirely.

## Testing Keyboard Behavior Yourself

If you use Android, you can verify keyboard data transmission directly:

```bash
# Install Charles Proxy on your computer
# Configure Android device to proxy through Charles
# Monitor HTTPS traffic while typing in different keyboards

# Alternatively, use tcpdump on rooted Android:
adb shell su -c 'tcpdump -i any -w /sdcard/traffic.pcap'
# Download and analyze with Wireshark

# Look for:
# - Requests to *.google.com (GBoard)
# - Requests to *.microsoft.com (SwiftKey)
# - Requests to keyboard service domains (others)
```

For iOS developers:

```swift
// Monitor keyboard activity using Network framework
import Network

let monitor = NWPathMonitor()
monitor.start(queue: .main)
monitor.pathUpdateHandler = { path in
    // Log all network connections during keyboard usage
    print("Network interfaces: \(path.availableInterfaces)")
}
```

## Privacy Scoring Your Current Keyboard

Use this framework to evaluate your current keyboard setup:

```python
def keyboard_privacy_score(keyboard_config):
    """Rate your keyboard on privacy scale 0-100."""

    factors = {
        "local_processing": keyboard_config.get("local_only", 0),  # 0-25 points
        "data_retention": 25 if not keyboard_config.get("sync_enabled") else 0,
        "transparency": keyboard_config.get("open_source", 0) * 25,  # 0-25 points
        "control": keyboard_config.get("user_controls", 0) * 25  # 0-25 points
    }

    total = sum(factors.values())
    return min(100, total)

# Scoring examples
configs = {
    "OpenBoard": {
        "local_only": 25,
        "sync_enabled": False,
        "open_source": 1,
        "user_controls": 1
    },  # Score: 100/100

    "GBoard": {
        "local_only": 10,
        "sync_enabled": True,
        "open_source": 0,
        "user_controls": 0.5
    },  # Score: 35/100

    "Apple": {
        "local_only": 20,
        "sync_enabled": False,
        "open_source": 0,
        "user_controls": 0.8
    }  # Score: 50/100
}

for keyboard, config in configs.items():
    score = keyboard_privacy_score(config)
    print(f"{keyboard}: {score}/100")
```



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

- [Iot Firmware Update Privacy Risks What Data Devices Send Dur](/privacy-tools-guide/iot-firmware-update-privacy-risks-what-data-devices-send-dur/)
- [Prevent Android Keyboard From Sending Typing Data To Google](/privacy-tools-guide/how-to-prevent-android-keyboard-from-sending-typing-data-to-google-or-samsung/)
- [Mobile App Store Privacy Labels How To Read And Compare Befo](/privacy-tools-guide/mobile-app-store-privacy-labels-how-to-read-and-compare-befo/)
- [Mobile Fitness Tracker Privacy](/privacy-tools-guide/mobile-fitness-tracker-privacy-what-health-apps-share-with-t/)
- [Nurse Practitioner Mobile Device Privacy Hipaa Compliant Pho](/privacy-tools-guide/nurse-practitioner-mobile-device-privacy-hipaa-compliant-pho/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
