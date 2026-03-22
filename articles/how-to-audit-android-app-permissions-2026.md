---
layout: default
date: 2026-03-21
last_modified_at: 2026-03-21
author: "Privacy Tools Guide"
title: "How to Audit Android App Permissions (2026)"
description: "Technical guide to auditing and restricting Android app permissions. Includes adb commands, permission groups, risk assessment, and practical examples."
permalink: /privacy-tools-guide/android-adb-app-permissions-audit/
categories: [security, guides]
tags: [privacy-tools-guide]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

### Step 1: How to Audit Android App Permissions (2026)

Android permissions control what data applications can access: location, contacts, camera, microphone, files, and more. Most users blindly grant permissions during installation, unaware of the privacy implications. This technical guide walks through auditing which permissions apps have, identifying risky configurations, and restricting permissions to minimize data exposure.

### Step 2: Android Permission Model Overview

Android 6.0+ (API 23+) uses runtime permissions. Apps can't access sensitive data without explicit user consent. Understanding permission categories helps identify privacy risks.

### Permission Groups

Android groups related permissions. Granting one permission in a group may provide access to others.

**Dangerous Permission Groups** (require explicit consent):

1. **CALENDAR**
 - READ_CALENDAR: View events
 - WRITE_CALENDAR: Create/modify events

2. **CAMERA**
 - CAMERA: Access camera hardware

3. **CONTACTS**
 - READ_CONTACTS: View all contacts
 - WRITE_CONTACTS: Modify contacts
 - GET_ACCOUNTS: View account list

4. **LOCATION**
 - ACCESS_FINE_LOCATION: GPS (precise, ~5-10 meter accuracy)
 - ACCESS_COARSE_LOCATION: Network location (approximate, ~100-1000 meter accuracy)
 - ACCESS_BACKGROUND_LOCATION: Location access when app not active

5. **MICROPHONE**
 - RECORD_AUDIO: Record audio

6. **PHONE**
 - CALL_PHONE: Make calls
 - READ_CALL_LOG: View call history
 - WRITE_CALL_LOG: Modify call history
 - USE_SIP: Make SIP calls
 - PROCESS_OUTGOING_CALLS: Monitor outgoing calls

7. **SMS**
 - SEND_SMS: Send text messages
 - RECEIVE_SMS: Receive text messages
 - READ_SMS: View text messages
 - RECEIVE_WAP_PUSH: Receive WAP push messages

8. **STORAGE**
 - READ_EXTERNAL_STORAGE: Read files on device
 - WRITE_EXTERNAL_STORAGE: Write files to device

9. **SENSORS**
 - BODY_SENSORS: Access accelerometer, gyroscope, etc.

**Normal Permissions** (auto-granted, less privacy risk):
- INTERNET
- ACCESS_NETWORK_STATE
- BLUETOOTH
- VIBRATE
- WAKE_LOCK
- These don't require explicit consent

## Prerequisites: Set Up ADB

Android Debug Bridge (adb) is the command-line tool for interacting with Android devices.

### Installation

**macOS**:
```bash
brew install android-platform-tools
adb version  # Verify installation
```

**Linux**:
```bash
sudo apt-get install adb
adb version
```

**Windows**:
Download Android SDK Platform Tools from Google, add to PATH, verify:
```bash
adb version
```

### Enable USB Debugging

1. Open **Settings** > **About Phone**
2. Tap **Build Number** 7 times until "Developer mode" appears
3. Go back to **Settings** > **Developer Options**
4. Enable **USB Debugging**
5. Connect phone via USB
6. Confirm authorization prompt on device

### Verify Connection

```bash
adb devices

# Output:
# List of attached devices
# ABC123XYZ  device
```

### Step 3: Method 1: Using adb to View All Permissions

### List All Installed Packages

```bash
adb shell pm list packages

# Output:
# package:com.google.android.apps.messaging
# package:com.whatsapp
# package:com.spotify.music
# package:com.example.myapp
```

**Filter results**:
```bash
# Only third-party apps (exclude system apps)
adb shell pm list packages -3

# Specific package
adb shell pm list packages | grep -i mail
```

### View Permissions for Specific App

```bash
adb shell pm dump com.whatsapp | grep -A 100 "requested permissions:"

# Output:
# requested permissions:
#     android.permission.CAMERA
#     android.permission.RECORD_AUDIO
#     android.permission.ACCESS_FINE_LOCATION
#     android.permission.READ_CONTACTS
#     android.permission.WRITE_EXTERNAL_STORAGE
```

### Check Which Permissions Are Actually Granted

```bash
adb shell pm list permissions -d -g

# This shows only "dangerous" permissions that are granted
```

**Detailed permission status**:
```bash
adb shell dumpsys package com.whatsapp | grep -A 50 "Permissions:"

# Output shows:
# - Permissions app requested
# - Permissions explicitly granted
# - Permissions denied
```

### Step 4: Method 2: Automated Permission Audit Script

Create an audit of all apps and their permissions:

```bash
#!/bin/bash
# audit-permissions.sh - Thorough Android permission audit

echo "Android Permission Audit - $(date)"
echo "========================================"
echo ""

# Get list of all third-party apps
PACKAGES=$(adb shell pm list packages -3)

# Dangerous permission groups to watch
DANGEROUS=(
    "android.permission.CAMERA"
    "android.permission.RECORD_AUDIO"
    "android.permission.ACCESS_FINE_LOCATION"
    "android.permission.ACCESS_BACKGROUND_LOCATION"
    "android.permission.READ_CONTACTS"
    "android.permission.READ_CALL_LOG"
    "android.permission.READ_SMS"
    "android.permission.SEND_SMS"
)

# Loop through each app
for package in $PACKAGES; do
    package_name=${package#package:}

    # Get permissions for this package
    perms=$(adb shell dumpsys package $package_name 2>/dev/null | grep -A 50 "Permissions:")

    # Check if this app has any dangerous permissions
    has_dangerous=false
    for perm in "${DANGEROUS[@]}"; do
        if echo "$perms" | grep -q "$perm"; then
            has_dangerous=true
            break
        fi
    done

    # If app has dangerous permissions, report it
    if [ "$has_dangerous" = true ]; then
        echo "App: $package_name"
        echo "Dangerous permissions granted:"
        for perm in "${DANGEROUS[@]}"; do
            if echo "$perms" | grep -q "$perm"; then
                echo "  - $perm"
            fi
        done
        echo ""
    fi
done
```

Run the audit:
```bash
chmod +x audit-permissions.sh
./audit-permissions.sh > audit_report.txt
```

### Step 5: Method 3: Using Android Studio

For developers, Android Studio provides GUI tools:

1. **Open Android Studio** > **Device File Explorer** (bottom right)
2. Navigate to `/data/system/users/0/runtime-permissions/`
3. Open `com.whatsapp.xml` (or any app)
4. View permission grants in XML format

### Step 6: Real-World Examples: Permission Analysis

### Example 1: Messaging App (WhatsApp)

```bash
adb shell dumpsys package com.whatsapp | grep "Permissions:" -A 30

# Typical output shows:
android.permission.INTERNET ✓ (Normal - auto-granted)
android.permission.CAMERA ✓ (Granted - for video calls)
android.permission.MICROPHONE ✓ (Granted - for calls/messages)
android.permission.READ_CONTACTS ✓ (Granted - contact sync)
android.permission.ACCESS_FINE_LOCATION ? (May be requested but not granted)
android.permission.WRITE_EXTERNAL_STORAGE ✓ (File sharing)
```

**Risk assessment**: CAMERA + MICROPHONE + CONTACTS = Very high data access. Messenger apps legitimately need these.

**Recommendation**: Grant, but monitor camera/mic access.

### Example 2: Weather App

```bash
adb shell dumpsys package com.weather.app | grep "Permissions:" -A 15

# Typical permissions:
android.permission.INTERNET ✓
android.permission.ACCESS_COARSE_LOCATION ✓ (Network location for weather)
android.permission.ACCESS_FINE_LOCATION ? (Don't need GPS precision)
```

**Risk assessment**: FINE_LOCATION not necessary for weather.

**Recommendation**: Deny ACCESS_FINE_LOCATION, allow ACCESS_COARSE_LOCATION only.

### Example 3: Social Media App (Instagram)

```bash
adb shell dumpsys package com.instagram.android | grep "Permissions:" -A 20

# Typical permissions:
android.permission.CAMERA ✓
android.permission.ACCESS_FINE_LOCATION ✓
android.permission.READ_CONTACTS ✓
android.permission.WRITE_EXTERNAL_STORAGE ✓
android.permission.RECORD_AUDIO ✓
android.permission.READ_CALENDAR ✓
android.permission.BODY_SENSORS ✓
android.permission.ACCESS_BACKGROUND_LOCATION ✓
```

**Risk assessment**: Excessive permissions (especially CALENDAR, BODY_SENSORS, BACKGROUND_LOCATION) for a photo-sharing app. Each permission represents data leakage risk.

**Recommendation**: Deny calendar, body sensors, and background location.

### Step 7: Revoke Permissions via ADB

After identifying risky permissions, revoke them.

### Revoke Single Permission

```bash
adb shell pm revoke --user 0 com.instagram.android android.permission.ACCESS_FINE_LOCATION

# Verify revocation:
adb shell dumpsys package com.instagram.android | grep "ACCESS_FINE_LOCATION"
```

### Revoke Multiple Permissions at Once

```bash
#!/bin/bash
# revoke-suspicious.sh

PACKAGE="com.instagram.android"
PERMS=(
    "android.permission.ACCESS_FINE_LOCATION"
    "android.permission.READ_CALENDAR"
    "android.permission.BODY_SENSORS"
    "android.permission.ACCESS_BACKGROUND_LOCATION"
)

for perm in "${PERMS[@]}"; do
    echo "Revoking $perm from $PACKAGE"
    adb shell pm revoke --user 0 $PACKAGE $perm
done

echo "Done. Verify in Settings > Apps > $PACKAGE > Permissions"
```

### Step 8: Permission Monitoring: Know When Apps Request Permissions

Once revoked, monitor when apps try to use revoked permissions.

### Monitor Permission Denials

```bash
adb logcat *:S ActivityManager:V | grep "Permission"

# Or grep for specific apps:
adb logcat | grep "com.instagram.android.*Permission"
```

### Watch for Permission Re-requests

Some apps repeatedly request permissions you've denied. Using a logcat filter:

```bash
adb logcat | grep "startActivityForResult\|onActivityResult"
```

### Step 9: Complete Audit Checklist

Use this checklist for regular privacy audits:

### Monthly Permission Audit

- [ ] List all installed third-party apps: `adb shell pm list packages -3`
- [ ] Check for new app installations
- [ ] Review any updates to existing apps
- [ ] Run permission audit script
- [ ] Identify and document suspicious permission grants

### Quarterly Dangerous Permission Review

For each dangerous permission group:

**CAMERA**:
- [ ] Does the app have legitimate camera use? (video call app, camera app, etc.)
- [ ] Is background camera access enabled? (Should be rare)

**LOCATION**:
- [ ] Fine vs coarse location: Does the app legitimately need GPS precision?
- [ ] Is background location enabled? (Most apps shouldn't have this)
- [ ] Check location access frequency via battery settings

**CONTACTS**:
- [ ] Does the app need contact read access? (Messaging/phone apps only)
- [ ] Is write_contacts necessary? (Rarely)

**MICROPHONE**:
- [ ] Legitimate audio recording use?
- [ ] Background recording? (Should be prevented)

**STORAGE**:
- [ ] Does app need external storage? (Photos/docs apps yes, others maybe not)
- [ ] Is write access necessary?

**SMS/CALL_LOG**:
- [ ] Only phone/messaging apps should have these
- [ ] Anything unusual?

### Quarterly App Removal

- [ ] Review unused apps
- [ ] Remove apps with excessive permissions
- [ ] Document reason for keeping apps with high permissions

### Step 10: Permission Risk Scoring

Quantify privacy risk by app:

```
Risk Score Calculation:

Each dangerous permission = 1 point
Background location = 2 points (especially risky)
Multiple sensors (calendar + location + contacts + mic) = +1 bonus

Example: WhatsApp
- CAMERA: 1 point
- MICROPHONE: 1 point
- CONTACTS: 1 point
- STORAGE: 1 point
Total: 4 points = MEDIUM risk (justified for messaging app)

Example: Instagram
- CAMERA: 1 point
- MICROPHONE: 1 point
- CONTACTS: 1 point
- LOCATION (fine): 1 point
- LOCATION (background): 2 points
- CALENDAR: 1 point
- BODY_SENSORS: 1 point
- STORAGE: 1 point
Total: 10 points = HIGH risk (not justified for photo app)

Risk categories:
0-3 points: LOW risk
4-6 points: MEDIUM risk
7-10 points: HIGH risk
11+ points: VERY HIGH risk (consider uninstalling)
```

## Best Practices Summary

1. **Grant minimal permissions**: Only approve permissions when app requests them, deny unnecessary ones
2. **Monitor granted permissions**: Review Settings > Apps > Permissions quarterly
3. **Prefer coarse over fine location**: Use network location when possible
4. **Disable background permissions**: Deny background location/camera unless essential
5. **Don't use app-specific folders**: Avoid WRITE_EXTERNAL_STORAGE if possible
6. **Check update behaviors**: New app versions may request new permissions
7. **Remove unused apps**: Uninstall apps with excessive permissions you don't use
8. **Use work profiles**: Separate work apps from personal, restrict permissions in each
9. **Regular audits**: Monthly quick review, quarterly detailed audit
10. **Document decisions**: Keep notes on why you granted certain permissions

### Step 11: Recommended Permissions by App Type

**Messaging Apps** (WhatsApp, Signal, Telegram):
- CAMERA ✓
- MICROPHONE ✓
- CONTACTS ✓
- STORAGE ✓
- INTERNET ✓
- LOCATION ✗ (deny)
- CALENDAR ✗ (deny)

**Navigation Apps** (Google Maps):
- LOCATION (fine) ✓
- LOCATION (background) ✓ (necessary for navigation)
- CAMERA ✗ (deny)
- CONTACTS ✗ (deny)
- MICROPHONE ✗ (deny)

**Social Media Apps** (Instagram, Twitter):
- CAMERA ✓ (for photos)
- MICROPHONE ✓ (for videos)
- STORAGE ✓ (for media)
- LOCATION ✗ (deny)
- CONTACTS ✗ (deny)
- CALENDAR ✗ (deny)
- BODY_SENSORS ✗ (deny)

**Banking Apps**:
- INTERNET ✓
- CAMERA ✓ (for mobile check deposits)
- All location permissions ✗
- All contacts/calendar ✗
- All microphone/sensors ✗

**Fitness Apps**:
- INTERNET ✓
- BODY_SENSORS ✓ (for activity tracking)
- LOCATION (coarse) ✓ (optional, for distance)
- MICROPHONE ✗ (deny)
- CONTACTS ✗ (deny)
- CALENDAR ✗ (deny)

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to audit android app permissions (2026)?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Audit Android App Permissions with ADB](/privacy-tools-guide/android-adb-app-permissions-audit)
- [Android App Permissions Audit Guide 2026](/privacy-tools-guide/android-app-permissions-audit-guide-2026/)
- [How To Audit Android App Permissions And Revoke Unnecessary](/privacy-tools-guide/how-to-audit-android-app-permissions-and-revoke-unnecessary-/)
- [How to Audit Android App Permissions: Step-by-Step Guide](/privacy-tools-guide/how-to-audit-android-app-permissions-guide/)
- [Android Storage Scopes How Modern Permissions Limit App Acce](/privacy-tools-guide/android-storage-scopes-how-modern-permissions-limit-app-acce/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
