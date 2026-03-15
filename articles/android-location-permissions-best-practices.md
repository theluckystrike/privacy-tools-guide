---

layout: default
title: "Android Location Permissions Best Practices"
description: "A comprehensive guide to implementing Android location permissions correctly. Learn about runtime permissions, foreground vs background location, and privacy-preserving strategies for developers."
date: 2026-03-15
author: theluckystrike
permalink: /android-location-permissions-best-practices/
categories: [guides]
---

{% raw %}

Location access is one of the most sensitive permissions an Android app can request. Users entrust apps with their physical whereabouts, and mishandling this data damages trust and violates platform policies. This guide covers the essential patterns for requesting, handling, and managing location permissions in modern Android development.

## Understanding Location Permission Types

Android distinguishes between several location permission types, each with different privacy implications and user experience requirements.

### Fine Location

Fine location provides precise GPS coordinates. It requires the `ACCESS_FINE_LOCATION` permission and offers accuracy within a few meters. Use this when your app needs exact positioning, such as navigation apps, fitness trackers, or location-based AR experiences.

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

### Coarse Location

Coarse location uses Wi-Fi and cellular networks to determine position within a larger area (typically 100-1000 meters). Request this when approximate location suffices, such as local news apps showing neighborhood weather or store finders.

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

### Foreground vs Background Location

Since Android 10 (API level 29), background location requires a separate permission. Background location access is restricted and triggers an additional approval dialog after users grant foreground access. Most apps function perfectly with foreground-only location, which remains active while the app is visible on screen.

```xml
<!-- Foreground location (automatically granted with FINE or COARSE) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- Background location (requires additional runtime request) -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

## Runtime Permission Requests

Android 6.0 (API level 23) introduced runtime permissions. Users can revoke permissions at any time through system settings, so apps must handle permission denials gracefully.

### Requesting Foreground Location

Always check permission status before requesting. Use the Activity Result API for cleaner code:

```kotlin
private val locationPermissionRequest = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    when {
        permissions.getOrDefault(Manifest.permission.ACCESS_FINE_LOCATION, false) -> {
            // Precise location access granted
            startLocationUpdates()
        }
        permissions.getOrDefault(Manifest.permission.ACCESS_COARSE_LOCATION, false) -> {
            // Only approximate location access granted
            startLocationUpdates()
        }
        else -> {
            // No location access granted
            showPermissionRationale()
        }
    }
}

fun requestLocation() {
    when {
        shouldShowRequestPermissionRationale(Manifest.permission.ACCESS_FINE_LOCATION) -> {
            // Explain why your app needs location before requesting
            showRationaleDialog()
        }
        else -> {
            locationPermissionRequest.launch(
                arrayOf(
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.ACCESS_COARSE_LOCATION
                )
            )
        }
    }
}
```

### Requesting Background Location

Background location requires users to have granted foreground location first. Android presents a distinct screen explaining why background access is necessary. Approach this carefully, as excessive background location requests trigger Play Store rejections.

```kotlin
fun requestBackgroundLocation() {
    // First ensure foreground permission is granted
    if (ContextCompat.checkSelfPermission(
            this,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
    ) {
        // Now request background location
        val backgroundPermissionRequest = registerForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { granted ->
            if (granted) {
                enableBackgroundTracking()
            }
        }
        
        backgroundPermissionRequest.launch(
            Manifest.permission.ACCESS_BACKGROUND_LOCATION
        )
    }
}
```

## Privacy-Preserving Strategies

### Principle of Least Privilege

Request only the permission level your feature genuinely requires. If coarse location fulfills your use case, don't request fine location. Users notice and distrust unnecessary permission requests.

```kotlin
// Choose the minimum permission needed for your use case
val permission = if (needsPreciseLocation) {
    Manifest.permission.ACCESS_FINE_LOCATION
} else {
    Manifest.permission.ACCESS_COARSE_LOCATION
}
```

### Degrade Gracefully

Build features that function without location access. Show appropriate messaging rather than blocking functionality:

```kotlin
fun getUserLocation(): LiveData<Location?> {
    val result = MutableLiveData<Location?>()
    
    if (hasLocationPermission()) {
        fusedLocationClient.lastLocation.addOnSuccessListener { location ->
            result.value = location
        }
    } else {
        // Provide alternative experience
        result.value = null
        showLocationRequestPrompt()
    }
    
    return result
}
```

### Minimize Location Collection

Collect location data only when necessary and stop promptly when done. Use the appropriate location class for your accuracy requirements:

```kotlin
val locationRequest = LocationRequest.Builder(
    Priority.PRIORITY_HIGH_ACCURACY,
    10000L // 10 second interval
).apply {
    setMinUpdateIntervalMillis(5000L)
    setWaitForAccurateLocation(false)
}.build()
```

## Handling Permission Denials

Users frequently deny location permissions. Your app must handle this gracefully without degrading the entire experience.

```kotlin
private fun handleLocationPermissionDenied() {
    val preferences = getSharedPreferences("permission_tracking", MODE_PRIVATE)
    val deniedCount = preferences.getInt("location_denied_count", 0)
    
    when {
        deniedCount >= 2 -> {
            // User has denied multiple times
            // Direct them to settings with a clear explanation
            showSettingsRedirectDialog(
                message = "Location access is needed for this feature. Please enable it in Settings."
            )
        }
        shouldShowRequestPermissionRationale(Manifest.permission.ACCESS_FINE_LOCATION) -> {
            // Show educational UI explaining the benefit
            showBenefitExplanation()
        }
        else -> {
            // First denial - show friendly reminder
            showCasualPrompt()
        }
    }.also {
        preferences.edit().putInt("location_denied_count", deniedCount + 1).apply()
    }
}
```

## Testing Location Permissions

Verify your permission handling across scenarios. Use Android Emulator's location controls to simulate different scenarios:

```bash
# Send mock location via ADB
adb emu geo fix 37.7749 -122.4194  # San Francisco coordinates
```

Test these scenarios:
- Fresh install with permission grant
- Permission denial on first request
- Permission revocation from settings
- Background location scenarios
- Multiple denial cycles

## Platform Compliance

Google Play policies strictly regulate location permissions. Background location access requires a prominent in-app disclosure and is reviewed manually. Avoid requesting background location unless your app's core functionality genuinely depends on it—such as fitness tracking apps, family safety utilities, or delivery services tracking drivers.

Frequent location updates in the background trigger battery drain warnings and potential Play Store warnings. Use geofencing APIs for event-driven location needs instead of continuous polling.

## Summary

Android location permissions require careful implementation balancing functionality with user privacy. Remember these key principles:

1. Request only the minimum accuracy level your feature needs
2. Always check permission status before accessing location
3. Handle denials gracefully without blocking app usage
4. Avoid background location unless essential
5. Provide clear explanations when requesting sensitive permissions

Proper location permission handling builds user trust and ensures Play Store compliance. Users appreciate transparency about why their location data matters to your app's functionality.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
