---
layout: default
title: "Audit Android App Permissions with ADB"
description: "Use ADB to audit and revoke Android app permissions from your computer. Find apps with excessive access to location, camera, and contacts."
date: 2026-03-21
author: theluckystrike
permalink: android-adb-app-permissions-audit
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Android's Settings menu shows permissions per app, but it's slow and incomplete — it doesn't show you which apps have which permission groups in bulk. ADB (Android Debug Bridge) lets you query and modify permissions across all installed apps from your computer in seconds.

This guide covers extracting a full permissions audit, finding problematic grants, and revoking permissions without uninstalling apps.

## Prerequisites

### Enable USB Debugging on Android

1. Settings > About Phone > tap "Build number" 7 times (enables Developer Options)
2. Settings > Developer Options > USB debugging > Enable
3. Connect phone to computer via USB

### Install ADB on Your Computer

```bash
# Ubuntu/Debian
sudo apt install adb

# macOS
brew install android-platform-tools

# Windows — download Platform Tools from:
# https://developer.android.com/studio/releases/platform-tools
# Add to PATH after extracting
```

Verify connection:

```bash
adb devices
```

Output should show your device:
```
List of devices attached
R5CNA1234B   device
```

If it shows "unauthorized", check your phone for the "Allow USB debugging" prompt and accept it.

## List All Installed Apps

Get a complete list of installed package names:

```bash
# All packages
adb shell pm list packages

# Third-party (non-system) apps only
adb shell pm list packages -3

# System apps only
adb shell pm list packages -s

# Include APK path
adb shell pm list packages -f -3
```

To map package names to human-readable app names:

```bash
# For each package, get the label
adb shell pm list packages -3 | sed 's/package://' | while read pkg; do
  label=$(adb shell dumpsys package "$pkg" 2>/dev/null | grep "versionName" | head -1)
  echo "$pkg: $label"
done
```

Or use this one-liner to get app labels via dumpsys:

```bash
adb shell cmd package list packages -3 | cut -d: -f2 | while IFS= read -r pkg; do
  name=$(adb shell dumpsys package "$pkg" 2>/dev/null | grep "applicationInfo" | grep "label=" | sed 's/.*label="\([^"]*\)".*/\1/' | head -1)
  echo "$pkg ($name)"
done
```

## Dump All Permissions for All Apps

Get a complete permissions dump:

```bash
# Full package info for all installed apps
adb shell dumpsys package > /tmp/packages_full.txt

# Extract just the granted permissions
grep -A100 "^Package \[" /tmp/packages_full.txt | grep -E "(Package \[|grantedPermissions|android.permission)" > /tmp/permissions_summary.txt
```

For a cleaner per-app permissions list:

```bash
adb shell pm list packages -3 | sed 's/package://' | while read pkg; do
  echo "=== $pkg ==="
  adb shell dumpsys package "$pkg" | grep -E "android.permission\.[A-Z_]+" | grep "granted=true"
  echo ""
done > /tmp/app_permissions.txt
```

View the results:

```bash
cat /tmp/app_permissions.txt | grep -B2 "LOCATION\|CAMERA\|MICROPHONE\|CONTACTS\|READ_SMS\|CALL_LOG"
```

## Find Specific High-Risk Permissions

Search for apps with specific sensitive permissions:

```bash
# Apps with location access
adb shell dumpsys package | grep -B20 "android.permission.ACCESS_FINE_LOCATION" | grep "Package \["

# Apps with camera access
adb shell dumpsys package | grep -B20 "android.permission.CAMERA" | grep "Package \["

# Apps with microphone access
adb shell dumpsys package | grep -B20 "android.permission.RECORD_AUDIO" | grep "Package \["

# Apps with contacts access
adb shell dumpsys package | grep -B20 "android.permission.READ_CONTACTS" | grep "Package \["

# Apps that can read SMS (very sensitive)
adb shell dumpsys package | grep -B20 "android.permission.READ_SMS" | grep "Package \["

# Apps with call log access
adb shell dumpsys package | grep -B20 "android.permission.READ_CALL_LOG" | grep "Package \["
```

## Check a Specific App's Permissions

Inspect exactly what one app has been granted:

```bash
# Replace com.example.app with the actual package name
adb shell dumpsys package com.example.app | grep -A200 "grantedPermissions"
```

Or use the cleaner appops approach:

```bash
# See current permission states for an app
adb shell appops get com.example.app
```

Example output:
```
Package com.example.app:
  CAMERA: allow
  RECORD_AUDIO: allow
  ACCESS_FINE_LOCATION: allow
  READ_CONTACTS: ignore
  ...
```

## Revoke Permissions via ADB

Revoke a specific permission without uninstalling the app:

```bash
# Syntax: adb shell pm revoke <package> <permission>

# Revoke location from a weather app
adb shell pm revoke com.weather.example android.permission.ACCESS_FINE_LOCATION
adb shell pm revoke com.weather.example android.permission.ACCESS_COARSE_LOCATION

# Revoke microphone from a utility app
adb shell pm revoke com.utility.example android.permission.RECORD_AUDIO

# Revoke contacts access
adb shell pm revoke com.social.example android.permission.READ_CONTACTS
adb shell pm revoke com.social.example android.permission.WRITE_CONTACTS
```

Revoke multiple permissions from an app at once:

```bash
# Revoke all location permissions from a specific app
for perm in ACCESS_FINE_LOCATION ACCESS_COARSE_LOCATION ACCESS_BACKGROUND_LOCATION; do
  adb shell pm revoke com.suspicious.app "android.permission.$perm"
  echo "Revoked $perm from com.suspicious.app"
done
```

## Use appops for Finer Control

`appops` gives you more granular control than `pm revoke` — you can set a permission to "ignore" (silently fail) rather than fully revoking it (which can cause app crashes or popups):

```bash
# Ignore (silently deny) instead of revoke
adb shell appops set com.example.app CAMERA ignore

# Operations you can control
# CAMERA, RECORD_AUDIO, READ_CONTACTS, WRITE_CONTACTS
# ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION, ACCESS_BACKGROUND_LOCATION
# READ_CALL_LOG, WRITE_CALL_LOG, READ_SMS, SEND_SMS
# PROCESS_OUTGOING_CALLS

# Reset to default (prompts user again next time)
adb shell appops set com.example.app CAMERA default
```

## Bulk Audit Script

Save this script as `audit_permissions.sh`:

```bash
#!/bin/bash
HIGH_RISK_PERMS=(
  "ACCESS_FINE_LOCATION"
  "ACCESS_COARSE_LOCATION"
  "ACCESS_BACKGROUND_LOCATION"
  "CAMERA"
  "RECORD_AUDIO"
  "READ_CONTACTS"
  "WRITE_CONTACTS"
  "READ_SMS"
  "SEND_SMS"
  "READ_CALL_LOG"
  "WRITE_CALL_LOG"
  "READ_EXTERNAL_STORAGE"
  "WRITE_EXTERNAL_STORAGE"
)

echo "=== High-Risk Permission Audit ==="
echo "Generated: $(date)"
echo ""

for perm in "${HIGH_RISK_PERMS[@]}"; do
  echo "--- Apps with android.permission.$perm ---"
  adb shell dumpsys package 2>/dev/null | \
    grep -B30 "android.permission.$perm" | \
    grep "Package \[" | \
    sed 's/.*Package \[\(.*\)\].*/\1/'
  echo ""
done
```

Run it:

```bash
chmod +x audit_permissions.sh
./audit_permissions.sh > permission_report.txt
cat permission_report.txt
```

## Monitor App Network Activity

Combine permission audit with network monitoring:

```bash
# See which apps have open network connections
adb shell cat /proc/net/tcp6 | awk '{print $3}' | while read hex; do
  port=$(printf "%d" 0x${hex##*:})
  echo $port
done | sort -u

# Or use the more readable approach
adb shell dumpsys netstats | grep -E "iface=|uid=" | head -50
```

## Re-enable Debugging After Audit

When finished:

```bash
# Disconnect ADB
adb disconnect

# On phone: Settings > Developer Options > Disable USB debugging
# Or: Settings > Developer Options (toggle off entirely)
```

Leaving USB debugging enabled is a security risk — a malicious charger or USB cable could silently connect.

## Related Reading

- [Android App Permissions Audit Guide 2026](/android-app-permissions-audit-guide-2026)
- [Android ADB Commands for Removing Bloatware](/android-adb-commands-for-removing-bloatware-that-tracks-user)
- [Android Background Location Access](/android-background-location-access-which-apps-track-you-when)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
