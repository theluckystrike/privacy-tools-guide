---
layout: default
title: "Android App Permissions Audit Guide 2026"
description: "A guide to auditing Android app permissions in 2026. Learn how to review, manage, and control app access using ADB, Play Store insights"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /android-app-permissions-audit-guide-2026/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Android app permissions form the frontline defense between your personal data and the applications you install. With modern Android versions introducing more granular permission controls and new runtime permission categories, understanding how to audit these permissions has become essential for developers and power users who value their privacy and security.

This guide covers practical methods for auditing Android app permissions using built-in tools, ADB commands, and programmatic approaches. You'll learn how to identify overprivileged apps, understand permission groups, and implement automated audit workflows.

## Android Permission System Overview

Android organizes permissions into three protection levels that determine how users and developers interact with them:

- Normal permissions Granted automatically at install time (e.g., `INTERNET`, `ACCESS_NETWORK_STATE`)
- Signature permissions Granted only if the app is signed with the same certificate as the declaring app
- Dangerous permissions Require runtime user approval and grant access to sensitive data (e.g., `READ_CONTACTS`, `ACCESS_FINE_LOCATION`, `CAMERA`, `RECORD_AUDIO`)

Android 14 and the 2026 platform updates have added new permission categories including `READ_MEDIA_VISUAL_USER_SELECTED` for more granular photo access and improved `NEARBY_WIFI_DEVICES` permission for device discovery without location dependency.

## Auditing Permissions via Settings

The most straightforward method for reviewing permissions uses Android's Settings interface. Navigate to **Settings > Apps**, select any application, and tap **Permissions** to see a complete list of requested permissions grouped by category.

For an overview of all apps with dangerous permissions, access **Settings > Privacy > Permission Manager**. This view groups permissions by category (Location, Camera, Microphone, Contacts, etc.) and shows which apps have access—essential for identifying apps that may no longer need certain permissions.

Modern Android versions display permission indicators in the status bar when apps actively use sensitive capabilities. A persistent camera or microphone icon warrants immediate investigation.

## Using ADB for Audits

The Android Debug Bridge (ADB) provides powerful command-line capabilities for auditing permissions at scale. Enable developer options and USB debugging on your device, then connect via USB or wireless debugging.

### Listing All App Permissions

Query all installed packages and their permissions using this command:

```bash
adb shell pm list packages -3 | while read pkg; do
  echo "=== $pkg ==="
  adb shell dumpsys package "$pkg" | grep -E "granted=|permission "
done
```

This outputs each third-party app with its permission status, including whether each dangerous permission is granted or denied.

### Exporting Permission Report

Generate a complete permission inventory for analysis:

```bash
adb shell pm list packages -3 -u > packages.txt
while read pkg; do
  pkg_trimmed=${pkg#package:}
  echo "Package: $pkg_trimmed" >> permissions_report.txt
  adb shell dumpsys package "$pkg_trimmed" | \
    grep -E "runtime" | \
    grep "permission" >> permissions_report.txt
done < packages.txt
```

### Revoking Permissions via ADB

For apps you cannot uninstall but want to restrict:

```bash
adb shell pm revoke com.example.app android.permission.CAMERA
adb shell pm revoke com.example.app android.permission.RECORD_AUDIO
adb shell pm revoke com.example.app android.permission.ACCESS_FINE_LOCATION
```

Note that some system apps may ignore revocations. You can also reset all permissions for an app:

```bash
adb shell pm reset-permissions com.example.app
```

## Programmatic Permission Auditing

For developers building security tools or implementing automated audits, Android provides the `PackageManager` API for inspecting permissions programmatically.

### Reading Permissions in Android Code

```kotlin
val packageManager = context.packageManager
val packageInfo = packageManager.getPackageInfo(
    "com.example.targetapp",
    PackageManager.GET_PERMISSIONS
)

packageInfo.requestedPermissions?.forEachIndexed { index, permission ->
    val isGranted = packageInfo.requestedPermissionsFlags[index] and
        PackageInfo.REQUESTED_PERMISSION_GRANTED != 0

    println("Permission: $permission")
    println("Granted: $isGranted")
}
```

### Permission Group Mapping

Dangerous permissions belong to permission groups. Understanding this relationship helps assess the actual access level:

```kotlin
val dangerousPermissions = mapOf(
    "android.permission.READ_CONTACTS" to "CONTACTS",
    "android.permission.WRITE_CONTACTS" to "CONTACTS",
    "android.permission.GET_ACCOUNTS" to "CONTACTS",
    "android.permission.ACCESS_FINE_LOCATION" to "LOCATION",
    "android.permission.ACCESS_COARSE_LOCATION" to "LOCATION",
    "android.permission.ACCESS_BACKGROUND_LOCATION" to "LOCATION",
    "android.permission.CAMERA" to "CAMERA",
    "android.permission.RECORD_AUDIO" to "MICROPHONE",
    "android.permission.READ_CALENDAR" to "CALENDAR",
    "android.permission.WRITE_CALENDAR" to "CALENDAR",
    "android.permission.READ_SMS" to "SMS",
    "android.permission.SEND_SMS" to "SMS"
)
```

Granting any permission within a group typically grants access to the entire group's data. For instance, `READ_CONTACTS` provides access to all contacts.

## Automating Permission Reviews

Creating a scheduled audit process helps maintain ongoing security. This Python script generates permission reports:

```python
import subprocess
import json
from datetime import datetime

def get_installed_packages():
    result = subprocess.run(
        ['adb', 'shell', 'pm', 'list', 'packages', '-3', '-f'],
        capture_output=True, text=True
    )
    packages = []
    for line in result.stdout.splitlines():
        pkg = line.replace('package:', '').split('=')[0]
        packages.append(pkg)
    return packages

def get_package_permissions(package):
    result = subprocess.run(
        ['adb', 'shell', 'dumpsys', 'package', package],
        capture_output=True, text=True
    )
    permissions = []
    for line in result.stdout.splitlines():
        if 'android.permission.' in line and 'granted=true' in line:
            perm = line.strip().split(':')[0]
            permissions.append(perm)
    return permissions

def generate_audit_report():
    packages = get_installed_packages()
    report = {
        'timestamp': datetime.now().isoformat(),
        'apps': []
    }

    for pkg in packages:
        try:
            perms = get_package_permissions(pkg)
            if perms:
                report['apps'].append({
                    'package': pkg,
                    'permissions': perms,
                    'count': len(perms)
                })
        except Exception as e:
            print(f"Error processing {pkg}: {e}")

    with open(f"permission_audit_{datetime.now().strftime('%Y%m%d')}.json", 'w') as f:
        json.dump(report, f, indent=2)

    return report

if __name__ == "__main__":
    report = generate_audit_report()
    print(f"Audit complete. Found {len(report['apps'])} apps with permissions.")
```

Run this weekly via cron or systemd timers to track permission changes over time.

## Interpreting Permission Requests

Not all permissions indicate malicious intent. When auditing, consider the app's legitimate functionality:

- Flashlight apps Require `CAMERA` (for flash control) and potentially `ACCESS_FINE_LOCATION` (incorrectly, often for ad targeting)
- Banking apps Require `READ_PHONE_STATE` for device identification, `CAMERA` for check deposits
- Messaging apps Require `READ_CONTACTS` to find connections, `RECORD_AUDIO` for voice messages
- Fitness apps Require `ACCESS_FINE_LOCATION` for workout tracking

Be suspicious of permission combinations that exceed the app's apparent function—a simple calculator requesting `READ_CONTACTS` or `RECORD_AUDIO` warrants investigation.

## Best Practices for Permission Hygiene

Implement these practices to maintain tight permission control:

1. Review before installing Play Store's permission summary shows all requested dangerous permissions before download
2. Use "While Using" restrictions Prefer "Allow only while using" over "Allow all the time" for location and camera
3. Deny non-essential permissions Many apps function without all requested permissions—test by denying
4. Audit quarterly Run permission audits every three months
5. Remove unused apps Uninstall applications no longer in use rather than leaving them installed
6. Check background access Android's Privacy Dashboard shows which apps accessed sensitive permissions recently


## Related Articles

- [Audit Android App Permissions with ADB](/privacy-tools-guide/android-adb-app-permissions-audit)
- [How to Audit Android App Permissions (2026)](/privacy-tools-guide/android-adb-app-permissions-audit/)
- [How To Audit Android App Permissions And Revoke Unnecessary](/privacy-tools-guide/how-to-audit-android-app-permissions-and-revoke-unnecessary-/)
- [How to Audit Android App Permissions: Step-by-Step Guide](/privacy-tools-guide/how-to-audit-android-app-permissions-guide/)
- [Android Storage Scopes How Modern Permissions Limit App Acce](/privacy-tools-guide/android-storage-scopes-how-modern-permissions-limit-app-acce/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
