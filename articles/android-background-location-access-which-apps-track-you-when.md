---

layout: default
title: "Android Background Location Access: Which Apps Track You."
description: "A technical guide for developers and power users on Android background location tracking. Learn which apps access your location when closed, how to."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /android-background-location-access-which-apps-track-you-when/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Many Android apps track your location in the background even when closed, including weather apps, fitness trackers, social media platforms, and navigation services that maintain background location permission. You can audit which apps access location with "Allow all the time" permission in Android Settings and deny background location access individually per app. Understanding how background location differs from foreground location and which apps are tracking you enables you to reclaim location privacy.

## Background Location vs Foreground Location

Android differentiates between foreground location access (when the app is actively displayed on screen) and background location access (when the app runs without visible interaction). Apps with background location permission can access your coordinates even when minimized, in the app drawer, or completely closed. This distinction matters because background tracking consumes significantly more data about your movements and often occurs without explicit user awareness.

When you grant "Allow all the time" location permission in Android 10 and later, you explicitly permit background location access. Android displays a warning dialog explaining that the app can access your location even when you're not using it—a transparency measure that many users approve without fully understanding the implications.

## Common Categories of Background Location Trackers

Several app categories exhibit particularly aggressive background location behavior:

**Weather applications** frequently maintain background location access to update forecasts based on your current position. While this provides genuine convenience, the continuous access generates detailed location histories that may extend beyond what users expect.

**Fitness and health apps** require background location for tracking runs, walks, and cycling sessions. These apps often accumulate years of location data creating comprehensive movement profiles.

**Social media platforms** represent some of the most aggressive background location collectors. Applications like Facebook, Instagram, and TikTok have faced scrutiny for accessing location data in the background for advertising targeting and user profiling purposes.

**Shopping and retail apps** increasingly request background location to monitor store visits, track shopping habits, and deliver location-based advertising. Major retail applications have been documented collecting location data even when users explicitly denied permission.

**Navigation and ride-sharing apps** naturally require background access for turn-by-turn directions and driver matching. However, these apps may retain location data long after the ride ends.

**News and content aggregation apps** sometimes request background location to deliver geographically targeted content and advertising, despite having no obvious location-dependent functionality.

## Technical Implementation of Background Tracking

Android provides developers with several APIs for implementing background location access:

```kotlin
// Request background location permission after foreground permissions
if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_BACKGROUND_LOCATION) 
    != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this, 
        arrayOf(Manifest.permission.ACCESS_BACKGROUND_LOCATION), 
        BACKGROUND_LOCATION_REQUEST_CODE)
}
```

The `ACCESS_BACKGROUND_LOCATION` permission requires that the app already holds either `ACCESS_FINE_LOCATION` or `ACCESS_COARSE_LOCATION`. Once granted, background location updates continue until explicitly revoked through device settings.

Location updates in the background arrive through the `LocationCallback` mechanism:

```kotlin
locationCallback = object : LocationCallback() {
    override fun onLocationResult(locationResult: LocationResult) {
        // Process location data
        val latitude = locationResult.lastLocation.latitude
        val longitude = locationResult.lastLocation.longitude
        
        // This executes even when app is not visible
        sendLocationToServer(latitude, longitude)
    }
}
```

Developers must declare the `android:foregroundServiceType="location"` permission in their manifest when requesting location updates from a foreground service—a requirement introduced in Android 10 to provide users visibility into actively running location services.

## Auditing Which Apps Track Your Location

Android provides built-in tools for auditing background location access:

1. **Privacy Dashboard** (Android 12+): Navigate to Settings > Privacy > Privacy Dashboard to view which apps recently accessed location data, including timestamps and access duration.

2. **Location Permissions Page**: Settings > Location > App Permission Settings displays each installed app with its current location permission level. Apps permitted for "Allow all the time" appear at the top with a location icon.

3. **Permission Manager**: Settings > Privacy > Permission Manager > Location provides granular control over individual app permissions.

For developers and power users seeking deeper insights, the `adb` command reveals precise permission states:

```bash
adb shell dumpsys package com.example.app | grep -A 10 "android.permission.ACCESS_BACKGROUND_LOCATION"
```

This command displays the permission grant state, last access timestamp, and frequency of background location access for any installed application.

## Practical Mitigation Strategies

Revoking background location access while maintaining foreground functionality represents the most practical balance between privacy and utility:

```kotlin
// Check if app has background location permission
fun hasBackgroundLocationPermission(context: Context): Boolean {
    return ContextCompat.checkSelfPermission(
        context, 
        Manifest.permission.ACCESS_BACKGROUND_LOCATION
    ) == PackageManager.PERMISSION_GRANTED
}
```

Users can downgrade permissions from "Allow all the time" to "Allow only while in use" through Settings > Location > App Permission Settings by selecting the specific app and choosing the appropriate option.

**Work Profile** creation provides additional isolation. Android work profiles create a separate container where apps operate with distinct permissions, allowing you to grant background location to work apps while restricting personal applications.

Third-party firewall applications like NetGuard or Island (for Samsung devices) can block network connections from specific applications, preventing location data from transmitting even when the app possesses background location permission.

For applications that genuinely require background location, reducing location accuracy to "Coarse location" rather than "Precise" limits the data collected while maintaining basic functionality:

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

## What Developers Need to Know

Building privacy-respecting location features requires intentional design decisions:

1. **Request background location as a secondary step** after establishing foreground necessity—users should understand why background access is needed before granting it.

2. **Provide clear explanations** in permission rationale dialogs describing exactly when and why background location will be used.

3. **Implement geofencing as an alternative** where possible—Android's Geofencing API triggers callbacks only when entering or exiting defined regions, reducing continuous tracking.

4. **Respect user revocation** by checking permission status before each background location operation and gracefully degrading functionality when access is removed.

5. **Minimize data retention** by processing location data locally or deleting it promptly rather than accumulating extensive histories.

## Detecting Hidden Location Access

Some apps have been documented requesting location permission but actually transmitting data through indirect methods:

- **WiFi scanning**: Even without location permission, apps can approximate location by scanning nearby WiFi networks and comparing against geolocation databases.

- **Bluetooth beacon detection**: Similar to WiFi scanning, BLE beacons in retail environments enable tracking without GPS permission.

- **Sensor-based location inference**: Barometer data combined with other sensors can reveal floor-level position information.

Disabling WiFi scanning and Bluetooth when not actively used prevents these alternative tracking vectors.

---

Understanding background location access empowers you to audit which apps track you when not open and implement practical restrictions. Regular permission reviews, careful app selection, and using Android's built-in privacy controls provide meaningful protection against unnecessary location surveillance while preserving functionality for applications that genuinely need it.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
