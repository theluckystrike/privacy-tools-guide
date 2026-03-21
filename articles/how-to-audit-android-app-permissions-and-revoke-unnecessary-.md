---
layout: default
title: "How To Audit Android App Permissions And Revoke Unnecessary"
description: "Learn how to audit Android app permissions, identify unnecessary access, and revoke risky permissions. This guide covers built-in tools, ADB commands"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-audit-android-app-permissions-and-revoke-unnecessary-/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Android's permission system serves as a critical security boundary between apps and your personal data. With over 70 permission types available to developers, understanding which permissions your installed apps actually require—and which ones they abuse—remains essential for maintaining mobile privacy. This guide walks through practical methods to audit app permissions, identify suspicious access patterns, and revoke unnecessary permissions using both built-in Android features and advanced command-line tools.

## Understanding Android Permission Categories and Protection Levels

Android organizes permissions into protection levels that determine how apps can request and obtain access to sensitive resources. The protection level system forms a hierarchy of increasingly sensitive access. Understanding these levels helps you evaluate whether an app's permission requests are legitimate.

**Normal Permissions** (Low Risk): Cover low-risk operations like changing the wallpaper, accessing vibration settings, reading battery statistics, or enabling/disabling WiFi. Android grants these automatically without user consent during installation. Examples include:
- `android.permission.ACCESS_NETWORK_STATE`
- `android.permission.VIBRATE`
- `android.permission.INTERNET`

**Signature Permissions** (Medium Risk): Allow apps to access data shared by other apps signed with the same certificate—typically used by apps from the same developer or integrated device manufacturer suites. Regular users cannot grant these permissions manually; they're controlled at the system level.

**Dangerous Permissions** (High Risk): Involve personal data access that could compromise privacy or device security:
- Contacts (READ_CONTACTS, WRITE_CONTACTS)
- Location (ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION)
- Camera and Microphone (CAMERA, RECORD_AUDIO)
- Storage (READ_EXTERNAL_STORAGE, WRITE_EXTERNAL_STORAGE)
- Phone status (READ_CALL_LOG, READ_SMS)
- Body sensors (BODY_SENSORS for fitness trackers)

These always require explicit user approval.

Android 6.0 (API level 23) introduced runtime permissions, meaning apps must request dangerous permissions at runtime rather than automatically receiving them during installation. However, apps can still request permissions they don't immediately need, and many users approve requests without careful consideration—studies show approval rates of 70%+ without reading explanations.

## Using Android's Built-in Permission Manager

Android provides native tools for reviewing and managing permissions without requiring technical expertise. These tools provide sufficient functionality for most privacy audits.

Navigate to **Settings > Apps**, select any app, and tap **Permissions** to see what access that specific app has been granted. This interface shows each permission's status: "Allowed," "Denied," or "Allowed only while in use." The status matters significantly—"Allowed only while in use" is more restrictive than "Allowed" and is preferred for sensitive permissions.

For an overview, open **Settings > Privacy > Permission Manager**. This dashboard groups permissions by category—Location, Camera, Microphone, Contacts, Storage—and lists every app with access to each category. Sort by permission count to identify apps requesting unusual numbers of permissions. Apps with 15+ granted permissions warrant scrutiny.

Android 12 and later includes a **Permissions dashboard** showing recent permission usage. Access it via **Settings > Privacy > Permission Manager > All permissions**, then tap the clock icon next to each permission. This reveals which apps accessed specific sensors recently, helping identify:
- Apps accessing microphone or camera without user visibility
- Apps accessing location in the background
- Unexpected sensor access from apps that shouldn't need it

This usage log is crucial because some apps request permissions but never actually use them—misleading users about their privacy exposure.

## Auditing Permissions via ADB

For developers and power users, Android Debug Bridge (ADB) provides command-line access to detailed permission information. Enable developer options on your device, then connect via USB debugging. Install ADB on your computer:

```bash
# Verify ADB installation
adb version

# List all installed packages
adb shell pm list packages
```

To examine permissions for a specific app, retrieve its package name and query its permissions:

```bash
# Find package name by app name
adb shell pm list packages | grep -i "appname"

# Query all permissions for a package
adb shell dumpsys package com.example.app | grep -A 10 "granted=true"
```

The `dumpsys` command reveals whether permissions are granted or denied, along with their protection level. For a complete permission audit across all installed apps, export the data to a file:

```bash
adb shell pm dump packages > permissions_audit.txt
```

This dump includes every app's requested permissions, granted permissions, and installation timestamp—useful for identifying apps that request excessive permissions during installation.

## Identifying Suspicious Permission Patterns

When auditing permissions, certain patterns warrant closer scrutiny. A flashlight app requesting camera access makes sense, but requesting contacts or microphone access raises concerns. A simple note-taking app should not need location or phone status permissions.

Build a baseline expectation for common app categories:

| App Category | Expected Permissions |
|--------------|---------------------|
| Navigation | Location, Storage |
| Messaging | Contacts, SMS |
| Camera/Photo | Camera, Storage |
| Fitness | Location, Activity Recognition |
| News/Reader | Network, Storage |

Apps requesting permissions outside their functional category may be overreaching. Check the app's privacy policy—required by Google Play—if the permission requests seem disproportionate to the app's purpose.

## Revoking Permissions Safely

Android allows revoking dangerous permissions for any app, regardless of whether the app targets older API versions. Use the built-in interface or ADB:

```bash
# Revoke a specific permission via ADB
adb shell pm revoke com.example.app android.permission.READ_CONTACTS

# Revoke all dangerous permissions for an app
adb shell pm revoke com.example.app android.permission.*
```

Revoking permissions may cause app malfunctions. If an app crashes after losing a permission, the app likely requires that permission for core functionality. Consider whether you genuinely need the app or whether a less permission-hungry alternative exists.

For apps targeting Android 11 (API 30) or higher, scoped storage limits app access to shared storage. However, apps can still request broad access via the MANAGE_EXTERNAL_STORAGE permission, which Google Play restricts for most apps. If an app unnecessarily requests this permission, deny it and look for alternatives.

## Automating Permission Audits

For users managing multiple devices or performing regular security audits, scripting the permission review process saves time. Here's a bash script that identifies apps with unusual permission combinations and exports detailed reports:

```bash
#!/bin/bash

# Android Permission Audit Script
# Usage: ./audit-permissions.sh > permission_report.txt

echo "=== Android Permission Audit Report ==="
echo "Generated: $(date)"
echo ""

# Count total apps and permissions
TOTAL_APPS=$(adb shell pm list packages -3 | wc -l)
echo "Total third-party apps: $TOTAL_APPS"
echo ""

# Function to audit specific app
audit_app() {
    local pkg=$1
    local app_name=$(adb shell dumpsys package "$pkg" | grep "  versionName" | cut -d= -f2)
    local perms=$(adb shell dumpsys package "$pkg" | grep "granted=true" | wc -l)

    if [ "$perms" -gt 5 ]; then
        echo "HIGH PERMISSION APP: $pkg ($app_name)"
        echo "  Granted permissions: $perms"
        adb shell dumpsys package "$pkg" | grep "granted=true" | sed 's/^/    /'
        echo ""
    fi
}

# Audit apps with more than 5 dangerous permissions
echo "=== Apps with Excessive Permissions ==="
for pkg in $(adb shell pm list packages -3 | cut -d: -f2); do
    audit_app "$pkg"
done

# Check for sensitive permission patterns
echo "=== Apps with Camera + Microphone (suspicious combo) ==="
for pkg in $(adb shell pm list packages -3 | cut -d: -f2); do
    has_camera=$(adb shell dumpsys package "$pkg" | grep -c "android.permission.CAMERA.*granted=true")
    has_mic=$(adb shell dumpsys package "$pkg" | grep -c "android.permission.RECORD_AUDIO.*granted=true")
    if [ "$has_camera" -eq 1 ] && [ "$has_mic" -eq 1 ]; then
        echo "  $pkg"
    fi
done

# Check location + clipboard permissions
echo ""
echo "=== Apps with Location + Clipboard Access ==="
for pkg in $(adb shell pm list packages -3 | cut -d: -f2); do
    has_location=$(adb shell dumpsys package "$pkg" | grep -c "android.permission.ACCESS.*LOCATION.*granted=true")
    has_clipboard=$(adb shell dumpsys package "$pkg" | grep -c "android.permission.READ_LOGS.*granted=true")
    if [ "$has_location" -eq 1 ] && [ "$has_clipboard" -eq 1 ]; then
        echo "  $pkg"
    fi
done
```

Save as `audit-permissions.sh`, make executable with `chmod +x`, and run to generate detailed permission reports.

## Understanding Permission Grant Techniques

Modern apps use sophisticated techniques to escalate permission requests. Understanding these patterns helps identify problematic apps:

**Feature Creep Requests**: Apps request permissions claiming they're for future features. A note-taking app might request calendar access "for reminders" that don't exist yet.

**Legitimacy Bundling**: Developers group mandatory permissions with optional ones. A flashlight app bundles camera access (necessary) with location access (unnecessary).

**Dependency Spoofing**: Some apps claim system permissions are required when they're actually optional. GPS navigation might claim contacts are needed for address autocomplete, when manual entry works fine.

Detecting these patterns requires checking the app's actual functionality against requested permissions. Use the Permission Manager's usage history to verify whether an app actually uses granted permissions.

## Permission Grant Timestamps and Tracking

Android 13+ stores permission grant timestamps in the system logs. Use these to identify when permissions were granted and track evolution:

```bash
# View permission grant history (Android 13+)
adb shell dumpsys package com.example.app | grep "grantedPerissions"

# Monitor runtime permission requests in real-time
adb shell pm dump com.example.app | grep -A 20 "runtime permissions:"
```

Applications sometimes request permissions months after installation, during seemingly innocent updates. Timestamp awareness helps identify suspicious permission escalation.

## System Apps and Bloatware

Pre-installed system apps often ship with permissions that cannot be fully revoked through standard interfaces. On rooted devices, ADB provides deeper control:

```bash
# Revoke permission from a system app
adb shell pm revoke --user 0 com.android.preinstalledapp android.permission.READ_CALL_LOG

# Disable an app entirely
adb shell pm disable-user --user 0 com.android.bloatware

# List all permissions for a system app to understand its full access
adb shell dumpsys package com.android.systemui | grep -A 50 "permissions:"
```

Non-rooted devices limit your options to disabling apps via settings or using Android's Restricted User features on supported devices. The Privacy Dashboard remains your primary tool for monitoring background access on non-rooted devices.

Consider using a custom ROM like **Lineage OS** or **GrapheneOS** (for Pixel devices) which provides granular permission control that stock Android doesn't offer. GrapheneOS, in particular, implements stricter scoped storage and improved permission isolation.

## Best Practices for Ongoing Permission Hygiene

Permission management requires ongoing attention, not an one-time audit. Adopt these habits:

**Pre-installation audit**: Review permissions before installing any app—the Play Store shows requested permissions before download. Search for user reviews mentioning unusual permission requests or data collection.

**Regular cleanup**: Delete apps you haven't used in 30 days rather than letting them accumulate. App accumulation increases your attack surface.

**Automatic reset**: Enable "Permissions automatically reset" in Android 12+ settings, which revokes permissions for unused apps after 3 months of non-use.

**Monthly reviews**: Check the Permission Manager monthly for unexpected permission grants. Sometimes app updates add new permissions without user notification.

**Update discipline**: Before installing app updates, check the permissions list. Updates sometimes add suspicious new permissions—defer updates if permissions seem excessive.

For developers building Android apps, follow the principle of least privilege. Request permissions only when actually needed, explain specifically why your app needs each permission in the app store listing, and implement graceful degradation when users deny permissions. Users are more likely to grant requests from apps that transparently explain necessity.

## Permission Abuse Patterns

Knowing how developers abuse permissions helps you identify risky apps:

**Unnecessary Permission Stacking**: Apps request many overlapping permissions. A calendar app might request READ_CONTACTS, WRITE_CONTACTS, GET_ACCOUNTS, READ_CALENDAR, and WRITE_CALENDAR when only calendar functionality is used. Each extra permission expands attack surface.

**Background Execution Permissions**: Permissions like WAKE_LOCK, INTERNET, and RECEIVE_BOOT_COMPLETED allow apps to run in the background and autostart after device reboot. Legitimate uses exist, but many apps use these to maintain aggressive analytics.

**Package Visibility Escalation**: Apps use QUERY_ALL_PACKAGES to see which other apps you've installed. While some features legitimately need this (password managers, app launchers), it's also used to fingerprint users based on app inventory.

**Device Admin Requests**: Some apps request device admin privileges through the BIND_DEVICE_ADMIN permission, giving them extensive control including the ability to disable you from uninstalling them. Avoid granting device admin to any app except trusted security software.

Check these patterns when evaluating new apps. The presence of multiple red flags suggests the app is designed for data harvesting rather than core functionality.

## Android Privacy Distributions and Custom ROMs

For users wanting maximum permission control, custom Android distributions provide stronger privacy:

- **GrapheneOS** (Pixel devices only): Implements sandbox restrictions, disables unused sensors, and provides app-level permission control unavailable in stock Android
- **Lineage OS**: Open-source AOSP derivative with community security patches and privacy controls
- **Calyx OS**: Privacy-focused Lineage variant with built-in Tor support and minimal tracking
- **DivestOS**: Hardened AOSP with expanded privacy controls

These ROMs require technical comfort with flashing, but provide significantly better permission granularity and control than stock Android.



## Related Articles

- [Audit Android App Permissions with ADB](/privacy-tools-guide/android-adb-app-permissions-audit)
- [Android App Permissions Audit Guide 2026](/privacy-tools-guide/android-app-permissions-audit-guide-2026/)
- [How to Audit Android App Permissions (2026)](/privacy-tools-guide/audit-android-app-permissions/)
- [How to Audit Android App Permissions: Step-by-Step Guide](/privacy-tools-guide/how-to-audit-android-app-permissions-guide/)
- [Android Storage Scopes How Modern Permissions Limit App Acce](/privacy-tools-guide/android-storage-scopes-how-modern-permissions-limit-app-acce/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
