---
title: "How to Audit Android App Permissions Privacy Guide 2026"
description: "Step-by-step guide to auditing Android app permissions. ADB commands, permission groups, dangerous permissions list, automation scripts, and mitigation"
author: Privacy Tools Guide
date: 2026-03-21
permalink: /privacy-tools-guide/how-to-audit-android-app-permissions-privacy-guide-2026/
reviewed: true
score: 9
voice-checked: true
intent-checked: true
tags: [privacy-tools-guide, privacy]---

{% raw %}

## Why Android Permissions Audit Matters

Android apps request permissions silently. Your weather app requests location. Your notes app requests camera. Most users tap "Allow" without reading what they're granting. This guide shows you how to audit which apps have which permissions, revoke suspicious ones, and monitor for new requests.

## Android Permission Categories

Android divides permissions into groups:

**Dangerous Permissions** (require explicit user approval)
- Location (fine/coarse)
- Camera
- Microphone
- Contacts
- Calendar
- Call logs
- SMS
- Phone state
- Body sensors
- Photos/media
- Files (read/write)

**Normal Permissions** (granted automatically)
- Internet access
- Access network state
- Access WiFi state
- Bluetooth
- NFC
- Vibration

**System Permissions** (for system apps only)
- Read call logs
- Write call logs
- System alert window

## Manual Audit via Settings

**Method 1: Check App-by-App**

Go to Settings → Apps & notifications → App permissions

You'll see:
```
Location
├── [List of apps]
Camera
├── [List of apps]
Microphone
├── [List of apps]
Contacts
└── [List of apps]
```

**Red flags to look for:**
- Weather app with location: EXPECTED (legitimate)
- Weather app with camera: RED FLAG (why?)
- Notes app with microphone: RED FLAG
- Social media with contacts: EXPECTED but risky (data mining)
- Utility app with calendar: RED FLAG

**For each suspicious app:**
1. Open Settings → Apps → [App name] → Permissions
2. Revoke: Location, Camera, Microphone, Contacts, Files
3. Keep enabled: Network, internet (usually needed)
4. Test the app — does it still work?

**Example: Facebook Audit**

Facebook requests:
- Location (used for targeted ads) — REVOKE if you don't want tracking
- Camera (for video calls) — Keep or revoke depending on use
- Microphone (for calls) — Keep or revoke
- Contacts (friend suggestions) — REVOKE to prevent data mining
- Photos/media (share media) — Keep only Photos, revoke write access
- Phone state (call screening) — REVOKE

After revoking, Facebook still works for messages and feed, but can't access private data.

## ADB Method: Complete Permission Audit

For an audit, use Android Debug Bridge (ADB).

**Setup ADB:**

1. Install Android SDK: https://developer.android.com/studio
2. Enable Developer Mode on phone:
 - Settings → About phone → Build number (tap 7 times)
3. Enable USB Debugging:
 - Settings → Developer options → USB debugging (enable)
4. Connect phone via USB cable
5. Run: `adb devices` (should list your phone)

**List all apps with their permissions:**

```bash
adb shell pm list packages -3
```

Output:
```
package:com.facebook.katana
package:com.twitter.android
package:com.snapchat.android
package:com.whatsapp
...
```

The `-3` flag shows only user-installed apps (excludes system apps).

**Get permissions for a specific app:**

```bash
adb shell dumpsys package com.facebook.katana | grep -A 100 "requested permissions:"
```

Output:
```
requested permissions:
  android.permission.INTERNET
  android.permission.CAMERA
  android.permission.ACCESS_FINE_LOCATION
  android.permission.READ_CONTACTS
  android.permission.WRITE_CALENDAR
  android.permission.READ_CALL_LOG
  ...
```

**Check which permissions are actually GRANTED:**

```bash
adb shell dumpsys package com.facebook.katana | grep "granted permissions:"
```

Output:
```
granted permissions:
  android.permission.INTERNET
  android.permission.CAMERA
  android.permission.ACCESS_FINE_LOCATION
```

Notice: READ_CONTACTS is requested but NOT granted (you revoked it). Good.

## Automated Permission Audit Script

Create this Python script to audit all apps:

```python
#!/usr/bin/env python3
import subprocess
import json
from collections import defaultdict

# Dangerous permissions to monitor
DANGEROUS_PERMS = [
    'android.permission.ACCESS_FINE_LOCATION',
    'android.permission.ACCESS_COARSE_LOCATION',
    'android.permission.CAMERA',
    'android.permission.RECORD_AUDIO',
    'android.permission.READ_CONTACTS',
    'android.permission.WRITE_CONTACTS',
    'android.permission.READ_CALENDAR',
    'android.permission.WRITE_CALENDAR',
    'android.permission.READ_SMS',
    'android.permission.SEND_SMS',
    'android.permission.READ_CALL_LOG',
    'android.permission.WRITE_CALL_LOG',
    'android.permission.READ_PHONE_STATE',
    'android.permission.READ_EXTERNAL_STORAGE',
    'android.permission.WRITE_EXTERNAL_STORAGE',
]

def get_packages():
    """Get list of all user-installed packages"""
    result = subprocess.run(['adb', 'shell', 'pm', 'list', 'packages', '-3'],
                          capture_output=True, text=True)
    packages = [line.replace('package:', '').strip()
                for line in result.stdout.strip().split('\n')
                if line.startswith('package:')]
    return packages

def get_granted_perms(package):
    """Get list of granted dangerous permissions for package"""
    cmd = f"adb shell dumpsys package {package} | grep -A 200 'granted permissions:' | head -50"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    granted = []
    for line in result.stdout.split('\n'):
        line = line.strip()
        if 'android.permission.' in line:
            perm = line.replace('android.permission.', '').strip()
            granted.append(perm)

    return granted

def get_app_label(package):
    """Get human-readable app name"""
    cmd = f"adb shell dumpsys package {package} | grep -A 1 'android:label' | head -1"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    try:
        return result.stdout.strip().split('=')[1]
    except:
        return package

def main():
    print("Android Permission Audit Report")
    print("=" * 80)

    packages = get_packages()
    print(f"\nScanning {len(packages)} apps...\n")

    suspicious_apps = defaultdict(list)

    for package in packages:
        granted = get_granted_perms(package)
        dangerous_granted = [p for p in granted if f'android.permission.{p}' in DANGEROUS_PERMS]

        if dangerous_granted:
            app_label = get_app_label(package)
            suspicious_apps[package] = {
                'label': app_label,
                'permissions': dangerous_granted
            }

    # Print report
    print("APPS WITH DANGEROUS PERMISSIONS GRANTED:")
    print("-" * 80)

    for i, (package, info) in enumerate(sorted(suspicious_apps.items()), 1):
        print(f"\n{i}. {info['label']}")
        print(f"   Package: {package}")
        print(f"   Permissions:")
        for perm in info['permissions']:
            print(f"     - {perm}")

    # Summary
    print("\n" + "=" * 80)
    print(f"SUMMARY: {len(suspicious_apps)} apps have dangerous permissions granted")

    # Export to JSON for further analysis
    with open('permission_audit.json', 'w') as f:
        json.dump(suspicious_apps, f, indent=2)

    print(f"Full report saved to: permission_audit.json")

if __name__ == '__main__':
    main()
```

Run this script:
```bash
chmod +x audit_permissions.py
python3 audit_permissions.py
```

Output:
```
Android Permission Audit Report
================================================================================

Scanning 47 apps...

APPS WITH DANGEROUS PERMISSIONS GRANTED:---
-----------------------------------------------------------------------------

1. Facebook
 Package: com.facebook.katana
 Permissions:
 - ACCESS_FINE_LOCATION
 - CAMERA
 - RECORD_AUDIO
 - READ_CONTACTS

2. WhatsApp
 Package: com.whatsapp
 Permissions:
 - ACCESS_FINE_LOCATION
 - CAMERA
 - RECORD_AUDIO

3. Gmail
 Package: com.google.android.gm
 Permissions:
 - READ_CONTACTS
 - READ_CALENDAR

...
```

## Revoke Permissions via ADB

Once you identify suspicious permissions, revoke them:

```bash
adb shell pm revoke --user 0 com.facebook.katana android.permission.READ_CONTACTS
```

This revokes READ_CONTACTS from Facebook without uninstalling it.

**Batch revoke for a specific app:**

```bash
#!/bin/bash
APP="com.facebook.katana"

# Revoke all dangerous permissions from Facebook
adb shell pm revoke --user 0 $APP android.permission.ACCESS_FINE_LOCATION
adb shell pm revoke --user 0 $APP android.permission.CAMERA
adb shell pm revoke --user 0 $APP android.permission.RECORD_AUDIO
adb shell pm revoke --user 0 $APP android.permission.READ_CONTACTS
adb shell pm revoke --user 0 $APP android.permission.READ_CALENDAR

echo "Permissions revoked from $APP"
```

## Monitor Permission Changes Over Time

Track which permissions each app requests in each Android update:

```bash
# Baseline scan (run once per month)
python3 audit_permissions.py > /tmp/perms_$(date +%Y%m%d).txt

# Compare against previous month
diff /tmp/perms_20260221.txt /tmp/perms_20260321.txt
```

Expect:
- Some apps lose permissions (good — they got more secure)
- Some apps gain permissions (investigate why)
- New apps = new permission requests (review carefully)

## Permission Groups Explained

Android 6+ uses permission groups. Granting one permission grants all in the group:

**Location Group:**
- ACCESS_FINE_LOCATION (GPS, ~10m accuracy)
- ACCESS_COARSE_LOCATION (WiFi/cell, ~1km accuracy)

If you grant GPS, the app can also get WiFi-based location. You can't grant one without the other.

**Camera/Microphone Group:**
- CAMERA
- RECORD_AUDIO

Often requested together for video calls.

**Contacts Group:**
- READ_CONTACTS
- WRITE_CONTACTS

Both are usually requested for "friend suggestions" or "message contacts."

**Calendar Group:**
- READ_CALENDAR
- WRITE_CALENDAR

**Files Group (Android 11+):**
- READ_EXTERNAL_STORAGE
- WRITE_EXTERNAL_STORAGE

Note: Android 13 split this into more granular groups.

## Red Flag Permission Combinations

Certain permission combinations are suspicious:

**Social Media + Contacts + Microphone:**
- Indicates: aggressive data collection for advertising profiling

**Utility app + Location + Camera:**
- Indicates: spyware or data harvesting

**Notes app + Read Call Logs:**
- Indicates: sketchy app (no legitimate reason)

**Weather app + Contacts:**
- Indicates: data mining

**Game + SMS:**
- Indicates: premium SMS fraud risk

For each combination found, investigate the app's privacy policy or consider uninstalling.

## Check App Manifest

Advanced users can inspect the APK file directly:

```bash
# Extract APK
adb pull /data/app/com.facebook.katana/base.apk facebook.apk

# Decompile with apktool
apktool d facebook.apk

# View requested permissions
cat facebook/AndroidManifest.xml | grep "uses-permission"
```

Output:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.READ_CONTACTS" />
```

This shows what the app CAN request, not what it actually uses.

## Android 13+ Privacy Features

Newer Android versions added controls:

**Approximate Location:**
- Grant location but deny GPS accuracy
- Settings → Apps → [App] → Permissions → Location
- Select "Approximate" instead of "Precise"

**Clipboard Access Detection:**
- See when apps read your clipboard
- Settings → Privacy Dashboard → Clipboard access

**Mic/Camera Indicators:**
- Green dot appears when any app uses mic/camera
- Tap to see which app

**Hibernation:**
- Auto-revoke permissions from unused apps
- Settings → Apps → All apps → [App] → Permissions → Advanced
- Toggle "Remove permissions if app isn't used"

**Privacy Sandbox:**
- Google's replacement for tracking (still new)
- Toggle: Settings → Privacy → Ad privacy controls

## Monthly Permission Audit Checklist

Schedule this quarterly:

```
□ Run permission_audit.py
□ Compare results to previous quarter
□ Review new dangerous permissions granted
□ Identify apps requesting location unnecessarily
□ Revoke suspicious permissions
□ Check privacy policies of high-risk apps
□ Test each app still works after revoking permissions
□ Document any unexpected permission behavior
□ Update whitelist of trusted apps
□ Report findings in privacy journal
```

## Tools for Permission Auditing

**Graphene OS Permission Manager:**
- Part of Graphene OS (privacy-focused Android fork)
- Shows runtime permission access logs
- https://grapheneos.org

**Exodus Privacy:**
- Scans APKs for trackers
- Shows which SDKs are used
- https://exodus.dy.fi

**AppOps (Advanced):**
- Access via ADB: `adb shell cmd appops set [package] [permission] deny`
- Granular control over permissions
- Can deny permissions while still allowing app to function

**Privacy Dashboard (Android 12+):**
- Built-in dashboard of all permission usage
- Settings → Privacy Dashboard
- Shows which apps access what, when

## Final Recommendations

**Essential Revokes (do immediately):**
- Revoke location from apps that don't need it (Facebook, Twitter, TikTok)
- Revoke contacts from social apps
- Revoke call logs from any app requesting it
- Revoke camera from messaging apps (unless you use video calls)

**Safe Grants (acceptable):**
- Internet/network access (almost all apps need it)
- Vibration (harmless)
- WiFi state (harmless)
- Bluetooth (only if needed)

**Never Grant:**
- Read SMS from unknown apps
- Read call logs from non-phone apps
- Camera/mic from apps that don't obviously need it
- Contacts from apps not designed for messaging

Run your permission audit now. You'll likely find 3-5 apps with suspicious permissions.

## Frequently Asked Questions

**How long does it take to audit android app permissions privacy guide?**

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

- [Audit Android App Permissions with ADB](android-adb-app-permissions-audit)
- [Android App Permissions Audit Guide 2026](/android-app-permissions-audit-guide-2026/)
- [How to Audit Android App Permissions (2026)](/privacy-tools-guide/android-adb-app-permissions-audit/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

