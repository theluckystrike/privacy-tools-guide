---
layout: default
title: "How to Disable WiFi Scanning and Bluetooth Scanning on Android for Privacy"
description: "A comprehensive technical guide for developers and power users on disabling WiFi and Bluetooth scanning on Android to enhance privacy. Includes ADB commands, app configurations, and code-level solutions."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-disable-wifi-scanning-and-bluetooth-scanning-on-andro/
categories: [security, guides]
reviewed: false
score: 0
intent-checked: false
voice-checked: false
---

{% raw %}

Android devices continuously scan for nearby WiFi networks and Bluetooth devices to improve location accuracy and enable quick pairing features. While these scanning mechanisms provide convenience, they also create privacy concerns because they generate location data that can be collected by apps, network operators, and potentially malicious actors. This guide explains how to disable WiFi scanning and Bluetooth scanning on Android, targeting developers and power users who want granular control over their device's privacy settings.

## Understanding Android Scanning Behavior

Android performs WiFi scanning primarily to assist GPS positioning through WiFi-based positioning systems (WPS). When your device scans for networks, it records BSSIDs (MAC addresses of access points) along with signal strength data. This information gets uploaded to Google's location services and can be used to determine your approximate location even when GPS is disabled.

Bluetooth scanning works similarly, detecting nearby Bluetooth devices to enable features like Fast Pair, audio notifications, and location-based device discovery. Apps with location permissions can access this scanning data, creating potential privacy risks.

Both scanning types occur automatically in the background, often without explicit user notification. Disabling them requires a combination of system settings, developer options, and in some cases, ADB commands.

## Disabling WiFi Scanning Through System Settings

The most accessible method involves adjusting Android settings. On most devices running Android 10 or later, navigate to **Settings > Location > WiFi Scanning** and toggle the scanning option off. However, this setting alone may not completely disable all scanning behavior, as some apps can still trigger scans independently.

For more comprehensive control, access **Settings > Apps > Special app access > Location** and review which apps have location permissions. Disable location access for apps that don't require it, particularly utility apps, calculators, or file managers that shouldn't need your location.

On Samsung devices, additional controls exist under **Settings > Connections > WiFi** or **Settings > Privacy**, where you can disable "Improve location" and "Send WiFi diagnostic data" options.

## Disabling Bluetooth Scanning

Bluetooth scanning controls appear in **Settings > Connections > Bluetooth** on most devices. Toggle off any "Nearby device scanning" or "Bluetooth scanning" options. However, Android may still perform periodic scans for system features.

To fully disable Bluetooth scanning at the system level, you need Developer Options. Enable Developer Options by tapping the build number in **Settings > About phone** seven times, then access **Developer options > Networking > Bluetooth scanning** and disable the feature.

## Advanced Control Using ADB Commands

For complete control over scanning behavior, Android Debug Bridge (ADB) provides the most effective solution. Install ADB on your computer, enable USB debugging in Developer Options, and connect your device.

The following command disables WiFi scanning globally:

```bash
adb shell settings put global wifi_scan_always_enabled 0
```

To verify the setting:

```bash
adb shell settings get global wifi_scan_always_enabled
```

This command returns `0` if scanning is disabled or `1` if enabled. Note that this setting may reset after system updates or when resetting network settings.

For Bluetooth scanning, use:

```bash
adb shell settings put global ble_scan_always_enabled 0
```

Some Android versions require additional commands to prevent apps from triggering their own scans. You can revoke location permissions from specific apps:

```bash
adb shell pm revoke com.example.app android.permission.ACCESS_FINE_LOCATION
```

Replace `com.example.app` with the package name of the app you want to restrict.

## Application-Level Solutions for Developers

If you're developing an Android app, you can implement scanning detection and prevention in your own applications. Use the following code to detect when WiFi scanning occurs:

```kotlin
val wifiManager = applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager
val scanResults = wifiManager.scanResults
for (result in scanResults) {
    Log.d("WiFiScan", "Network: ${result.SSID}, BSSID: ${result.BSSID}")
}
```

To prevent your app from triggering scans, avoid calling `startScan()` directly and instead use the passive scanning API where available:

```kotlin
val wifiManager = applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager
// Request passive scan results without actively scanning
val scanResults = wifiManager.scanResults
```

Android 13 introduced additional restrictions on background location access. Request only the permissions your app genuinely needs:

```kotlin
if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) 
    != PackageManager.PERMISSION_GRANTED) {
    // Request minimal permission
    requestPermissions(arrayOf(Manifest.permission.ACCESS_COARSE_LOCATION), 1)
}
```

## Handling Location Services Alternatives

Disabling WiFi scanning affects location accuracy, particularly indoors where GPS signals weaken. Consider these alternatives to maintain functionality while preserving privacy:

- Use GPS-only location when precise tracking isn't required
- Employ VPN services that mask network identifiers
- Disable "Google Location Services" in **Settings > Location > Location Services** to prevent data uploads while maintaining basic location functionality

For developers building location-aware applications, consider using the Fused Location Provider, which intelligently manages power consumption and provides privacy-preserving location updates without requiring constant scanning.

## Testing Your Configuration

After implementing changes, verify scanning behavior using apps like "WiFi Analyzer" or "Bluetooth Scanner" from trusted developers. These apps display whether your device actively scans for networks or Bluetooth devices.

You can also monitor scanning behavior through Android's built-in logging:

```bash
adb logcat | grep -i "wifi_scan\|ble_scan"
```

If scanning continues despite your settings, check for apps with aggressive background behavior or system apps that may require separate configuration.

## Conclusion

Disabling WiFi scanning and Bluetooth scanning on Android requires a layered approach combining system settings, Developer Options, and potentially ADB commands. While complete disabling affects some convenience features like fast network switching and device pairing, the privacy benefits outweigh these tradeoffs for security-conscious users.

For developers, implementing privacy-conscious scanning practices in your applications demonstrates respect for user data and aligns with evolving privacy regulations. Always request only necessary permissions and consider using system-provided APIs that balance functionality with privacy protection.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
