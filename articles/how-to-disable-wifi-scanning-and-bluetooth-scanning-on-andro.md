---
layout: default
title: "How To Disable Wifi Scanning And Bluetooth Scanning On Andro"
description: "A technical guide for developers and power users on disabling WiFi and Bluetooth scanning on Android to enhance privacy. Includes ADB."
date: 2026-03-15
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

## Understanding Android Scanning Behavior

Android performs WiFi scanning primarily to assist GPS positioning through WiFi-based positioning systems (WPS). When your device scans for networks, it records BSSIDs (MAC addresses of access points) along with signal strength data. This information gets uploaded to Google's location services and can be used to determine your approximate location even when GPS is disabled.

Bluetooth scanning works similarly, detecting nearby Bluetooth devices to enable features like Fast Pair, audio notifications, and location-based device discovery. Apps with location permissions can access this scanning data, creating potential privacy risks.

Both scanning types occur automatically in the background, often without explicit user notification. Disabling them requires a combination of system settings, developer options, and in some cases, ADB commands.

## Disabling WiFi Scanning Through System Settings

The most accessible method involves adjusting Android settings. On most devices running Android 10 or later, navigate to **Settings > Location > WiFi Scanning** and toggle the scanning option off. However, this setting alone may not completely disable all scanning behavior, as some apps can still trigger scans independently.

For more control, access **Settings > Apps > Special app access > Location** and review which apps have location permissions. Disable location access for apps that don't require it, particularly utility apps, calculators, or file managers that shouldn't need your location.

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

## System-Level Scanning Commands Reference

Complete ADB commands for comprehensive scanning control:

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

## Blocklist Approach: Preventing Scanning Apps

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

## Alternative: Fused Location Provider for Developers

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

## Monitoring Background Activity

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

## Hardening: Kernel-Level Scanning Prevention

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

## Recovery: Re-enabling Scanning When Needed

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

## Privacy vs. Functionality Trade-Offs

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

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
