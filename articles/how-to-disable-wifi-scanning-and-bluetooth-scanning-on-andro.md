---
layout: default

permalink: /how-to-disable-wifi-scanning-and-bluetooth-scanning-on-andro/
description: "Follow this guide to how to disable wifi scanning and bluetooth scanning on andro with practical examples, tips, and step-by-step instructions for..."
tags: [privacy-tools-guide]
author: "Privacy Tools Guide"
reviewed: true
score: 9
date: 2026-03-15
categories: [guides]
---

layout: default
title: "How To Disable Wifi Scanning And Bluetooth Scanning On"
description: "A technical guide for developers and power users on disabling WiFi and Bluetooth scanning on Android to enhance privacy. Includes ADB"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-disable-wifi-scanning-and-bluetooth-scanning-on-andro/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Android devices continuously scan for nearby WiFi networks and Bluetooth devices to improve location accuracy and enable quick pairing features. While these scanning mechanisms provide convenience, they also create privacy concerns because they generate location data that can be collected by apps, network operators, and potentially malicious actors. This guide explains how to disable WiFi scanning and Bluetooth scanning on Android, targeting developers and power users who want granular control over their device's privacy settings.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced Control Using ADB Commands](#advanced-control-using-adb-commands)
- [Threat Model: Location Inference from Scanning](#threat-model-location-inference-from-scanning)
- [Advanced Scanning Detection and Prevention](#advanced-scanning-detection-and-prevention)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Android Scanning Behavior

Android performs WiFi scanning primarily to assist GPS positioning through WiFi-based positioning systems (WPS). When your device scans for networks, it records BSSIDs (MAC addresses of access points) along with signal strength data. This information gets uploaded to Google's location services and can be used to determine your approximate location even when GPS is disabled.

Bluetooth scanning works similarly, detecting nearby Bluetooth devices to enable features like Fast Pair, audio notifications, and location-based device discovery. Apps with location permissions can access this scanning data, creating potential privacy risks.

Both scanning types occur automatically in the background, often without explicit user notification. Disabling them requires a combination of system settings, developer options, and in some cases, ADB commands.

### Step 2: Disable WiFi Scanning Through System Settings

The most accessible method involves adjusting Android settings. On most devices running Android 10 or later, navigate to **Settings > Location > WiFi Scanning** and toggle the scanning option off. However, this setting alone may not completely disable all scanning behavior, as some apps can still trigger scans independently.

For more control, access **Settings > Apps > Special app access > Location** and review which apps have location permissions. Disable location access for apps that don't require it, particularly utility apps, calculators, or file managers that shouldn't need your location.

On Samsung devices, additional controls exist under **Settings > Connections > WiFi** or **Settings > Privacy**, where you can disable "Improve location" and "Send WiFi diagnostic data" options.

### Step 3: Disable Bluetooth Scanning

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

### Step 4: Application-Level Solutions for Developers

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

### Step 5: Handling Location Services Alternatives

Disabling WiFi scanning affects location accuracy, particularly indoors where GPS signals weaken. Consider these alternatives to maintain functionality while preserving privacy:

- Use GPS-only location when precise tracking isn't required
- Employ VPN services that mask network identifiers
- Disable "Google Location Services" in **Settings > Location > Location Services** to prevent data uploads while maintaining basic location functionality

For developers building location-aware applications, consider using the Fused Location Provider, which intelligently manages power consumption and provides privacy-preserving location updates without requiring constant scanning.

### Step 6: Test Your Configuration

After implementing changes, verify scanning behavior using apps like "WiFi Analyzer" or "Bluetooth Scanner" from trusted developers. These apps display whether your device actively scans for networks or Bluetooth devices.

You can also monitor scanning behavior through Android's built-in logging:

```bash
adb logcat | grep -i "wifi_scan\|ble_scan"
```

If scanning continues despite your settings, check for apps with aggressive background behavior or system apps that may require separate configuration.

## Threat Model: Location Inference from Scanning

Understanding why scanning creates privacy risks clarifies the value of disabling it:

**WiFi-Based Location**: When your device broadcasts WiFi scanning queries, it reveals the BSSIDs (MAC addresses) of nearby access points. Google's location services database maps these MAC addresses to physical locations with 10-50 meter accuracy. An attacker with access to your WiFi scans can infer your location without GPS.

**Temporal Patterns**: Regular WiFi scans at specific times and locations create movement patterns. Presence at a particular location every Tuesday at 3 PM suggests work commute or routine activity.

**Cross-Correlation with Other Data**: Combining WiFi scan metadata with other information (purchase location data from retailers, cell tower triangulation) enables higher-accuracy tracking.

**Bluetooth MAC Address Tracking**: Bluetooth devices transmit MAC addresses that don't randomize as frequently as WiFi. Persistent Bluetooth device discovery enables tracking as you move through environments.

Disabling scanning prevents this data generation entirely.

## Advanced Scanning Detection and Prevention

For developers building privacy-aware Android apps, detect and prevent excessive scanning:

```kotlin
// Advanced scanning detection and prevention
import android.content.Context
import android.net.wifi.WifiManager
import android.bluetooth.BluetoothManager
import android.util.Log

class ScanningMonitor(private val context: Context) {
    private val wifiManager = context.getSystemService(Context.WIFI_SERVICE) as WifiManager
    private val bluetoothManager = context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager

    fun detectActiveScanning(): ScanningStatus {
        val status = ScanningStatus()

        // Check WiFi scanning status
        try {
            val wifiResults = wifiManager.scanResults
            if (wifiResults.isNotEmpty()) {
                status.wifiScanActive = true
                status.wifiNetworksDetected = wifiResults.size
                Log.d("ScanMonitor", "Active WiFi scan detected: ${wifiResults.size} networks")
            }
        } catch (e: Exception) {
            Log.e("ScanMonitor", "Error checking WiFi: ${e.message}")
        }

        // Check Bluetooth scanning
        val bluetoothAdapter = bluetoothManager.adapter
        if (bluetoothAdapter?.isDiscovering == true) {
            status.bluetoothScanActive = true
            Log.d("ScanMonitor", "Active Bluetooth discovery in progress")
        }

        return status
    }

    fun preventBackgroundScanning() {
        // Implement app-specific scanning prevention
        // Request minimal location permissions
        // Disable background service scanning

        // For your app: don't call startScan()
        // Use passive results: wifiManager.scanResults

        Log.d("ScanMonitor", "Background scanning prevention enabled")
    }
}

data class ScanningStatus(
    var wifiScanActive: Boolean = false,
    var wifiNetworksDetected: Int = 0,
    var bluetoothScanActive: Boolean = false
)
```

### Step 7: System-Level Scanning Commands Reference

Complete ADB commands for scanning control:

```bash
# Disable all location-related scanning
adb shell settings put global wifi_scan_always_enabled 0
adb shell settings put global ble_scan_always_enabled 0
adb shell settings put global location_mode 0  # Disable location services entirely

# Revoke location permissions from all apps
adb shell pm revoke com.google.android.gms android.permission.ACCESS_FINE_LOCATION
adb shell pm revoke com.google.android.gms android.permission.ACCESS_COARSE_LOCATION

# Disable Google Location Services
adb shell settings put secure location_providers_allowed -gps,-network

# Disable WiFi scanning across all apps
adb shell settings put global wifi_scan_interval_ms 0

# Disable Bluetooth scanning
adb shell settings put global ble_scan_interval_ms 0

# Verify settings
adb shell settings get global wifi_scan_always_enabled
adb shell settings get global ble_scan_always_enabled

# Check location mode
adb shell settings get secure location_mode
```

### Step 8: Blocklist Approach: Preventing Scanning Apps

Alternative to system settings: block scanning at the network level.

```bash
#!/bin/bash
# Block known Google location services endpoints
# Run on your router or OPNsense firewall

# Google location services IPs
GOOGLE_LOCATION_IPS=(
    "64.18.0.0/20"       # Google location services
    "142.251.32.0/20"    # Google analytics
    "142.251.41.0/20"    # Google services
)

# Create firewall rules blocking these
for ip_range in "${GOOGLE_LOCATION_IPS[@]}"; do
    # For OPNsense:
    echo "pfctl -t google_tracking -T add $ip_range"

    # Then create a blocking rule for the google_tracking table
done

# Alternative: Block at DNS level
# Return NXDOMAIN for location services domains
cat >> /etc/unbound/unbound.conf.d/location-block.conf << 'EOF'
# Block WiFi scanning and location services
local-zone: "www.googleapis.com" redirect
local-data: "www.googleapis.com A 0.0.0.0"
local-zone: "maps.googleapis.com" redirect
local-data: "maps.googleapis.com A 0.0.0.0"
EOF
```

### Step 9: Alternative: Fused Location Provider for Developers

For apps that need location but want minimal scanning:

```kotlin
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationServices

class EfficientLocationProvider(private val context: Context) {
    private val fusedLocationClient: FusedLocationProviderClient =
        LocationServices.getFusedLocationProviderClient(context)

    fun getLocationWithMinimalScanning() {
        // Request coarse location (network-based, no WiFi scanning)
        val locationRequest = com.google.android.gms.location.LocationRequest.create().apply {
            priority = com.google.android.gms.location.LocationRequest.PRIORITY_LOW_POWER
            interval = 60000  // Update every 60 seconds, not continuously
            fastestInterval = 120000  // Never update faster than 2 minutes
        }

        // Use passive location listener - doesn't trigger active scans
        val callback = object : com.google.android.gms.location.LocationCallback() {
            override fun onLocationResult(result: com.google.android.gms.location.LocationResult) {
                for (location in result.locations) {
                    // Use location without continuous scanning
                    Log.d("Location", "Got passive location: ${location.latitude}")
                }
            }
        }

        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            callback,
            null  // Don't update on main thread
        )
    }
}
```

This approach provides location data while minimizing background scanning.

### Step 10: Monitor Background Activity

After disabling scanning, monitor to ensure apps aren't re-enabling it:

```bash
#!/bin/bash
# Monitor for unauthorized scanning enablement

# Get current scanning state
get_scanning_state() {
    echo "=== WiFi Scanning Status ==="
    adb shell settings get global wifi_scan_always_enabled

    echo "=== Bluetooth Scanning Status ==="
    adb shell settings get global ble_scan_always_enabled

    echo "=== Location Services Status ==="
    adb shell settings get secure location_mode
}

# Monitor logs for scanning attempts
monitor_scanning_attempts() {
    echo "Monitoring for scanning enablement attempts..."

    adb logcat | while read line; do
        if echo "$line" | grep -qi "wifi_scan\|ble_scan\|location.*enable"; then
            echo "⚠️  Scanning activity detected: $line"
        fi
    done
}

# Create hourly monitoring job
while true; do
    get_scanning_state
    sleep 3600  # Check every hour
done
```

### Step 11: Hardening: Kernel-Level Scanning Prevention

For the most security-conscious users with custom ROMs:

```bash
# Disable WiFi scanning at kernel level
adb shell "echo 0 > /sys/module/cfg80211/parameters/debug_scan"

# Check kernel parameters
adb shell "cat /proc/cmdline" | grep wifi

# For Lineage OS with optional kernel patches:
# Apply patches that disable background location scanning entirely
```

This requires root and custom ROM modifications, beyond typical user capabilities.

### Step 12: Recovery: Re-enabling Scanning When Needed

Some apps genuinely need location services for legitimate purposes:

```bash
# Re-enable WiFi scanning
adb shell settings put global wifi_scan_always_enabled 1

# Re-enable for specific app context
# Settings > Apps > [AppName] > Permissions > Location > Allow only while using the app

# Verify re-enablement
adb shell settings get global wifi_scan_always_enabled
```

Consider creating device profiles: one privacy-hardened configuration for personal use, another with normal permissions for navigation or location-based apps.

### Step 13: Privacy vs. Functionality Trade-Offs

Complete scanning disablement comes with costs:

**Loss of benefits**:
- WiFi positioning (GPS works, but slower indoors)
- Bluetooth device discovery (car audio, wearables)
- Location-based app features
- Ambient device sensing

**Workarounds**:
- Use GPS-only location mode when GPS signal available
- Manually pair Bluetooth devices instead of auto-discovery
- Accept reduced location accuracy for privacy
- Use alternative location services that don't require scanning

The choice depends on your threat model and location accuracy requirements.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to disable wifi scanning and bluetooth scanning on?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How to Tell if Your Bluetooth Is Being Intercepted](/how-to-tell-if-your-bluetooth-is-being-intercepted-nearby/)
- [Disable Location Services Completely On macOS While Keeping](/how-to-disable-location-services-completely-on-macos-while-keeping-apps-functional/)
- [How To Use Adb To Disable Android System Apps That Spy On](/how-to-use-adb-to-disable-android-system-apps-that-spy-on-yo/)
- [Android Privacy Best Practices 2026](/android-privacy-best-practices-2026/)
- [iPhone Privacy Settings Complete Guide Turn Off All Tracking](/iphone-privacy-settings-complete-guide-turn-off-all-tracking/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
```
```
{% endraw %}
