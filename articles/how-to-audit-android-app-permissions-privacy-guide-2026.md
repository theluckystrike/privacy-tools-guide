---
layout: default
title: "How to Audit Android App Permissions Privacy Guide 2026"
description: "Step-by-step guide to auditing and restricting Android app permissions for better privacy including ADB commands, permission managers, and automation"
date: 2026-03-22
last_modified_at: 2026-03-22
author: "Privacy Tools Guide"
permalink: /how-to-audit-android-app-permissions-privacy-guide-2026/
categories: [guides]
tags: [privacy-tools-guide, tools, privacy]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}
Why Android Permissions Audit Matters


Android apps request permissions silently. Your weather app requests location. Your notes app requests camera. Most users tap "Allow" without reading what they're granting. This guide shows you how to audit which apps have which permissions, revoke suspicious ones, and monitor for new requests.

Android Permissions - The Surveillance Surface

Android apps request permissions at install time. Most users tap "Allow" on all of them. This is a security failure that manufacturers and app developers exploit.

A weather app requests:
- Location (expected)
- Contacts (why?)
- Calendar (why?)
- Microphone (why?)
- Camera (why?)
- Call logs (why?)

Each permission is a surveillance channel. Location reveals where you live, work, and frequent. Contacts reveal your social graph. Microphone enables silent eavesdropping. Calendar reveals your schedule.

Android 6.0+ (API 23) introduced granular permissions: apps ask for permissions at runtime, not install-time. This is better, but apps still request excessive permissions. Users don't understand the implications. Apps justify them with vague language ("Needs location to improve service").

This guide shows you how to:
1. Audit which permissions apps have
2. Restrict permissions to minimum necessary
3. Automate permission monitoring
4. Validate that apps work with restricted permissions


Table of Contents

- [Why Android Permissions Audit Matters](#why-android-permissions-audit-matters)
- [Android Permissions - The Surveillance Surface](#android-permissions-the-surveillance-surface)
- [Prerequisites](#prerequisites)
- [Part 1 - Manual Permission Audit](#part-1-manual-permission-audit)
- [Part 2 - Restrict Permissions](#part-2-restrict-permissions)
- [Part 3 - Verify Restricted Apps Work](#part-3-verify-restricted-apps-work)
- [Part 4 - Automate Permission Monitoring](#part-4-automate-permission-monitoring)
- [Part 5 - Decision Framework](#part-5-decision-framework)
- [Part 6 - Automation via Device Policy](#part-6-automation-via-device-policy)
- [Dangerous Permission Categories Explained](#dangerous-permission-categories-explained)
- [Troubleshooting Restricted Apps](#troubleshooting-restricted-apps)
- [Privacy Best Practices Summary](#privacy-best-practices-summary)

Prerequisites

- Android device (6.0 Marshmallow or later)
- USB debugging enabled
- ADB (Android Debug Bridge) installed on computer
- Willingness to revoke permissions and test app behavior

Optional:
- Frida (dynamic instrumentation framework)
- logcat (built into ADB)

Part 1 - Manual Permission Audit

Step 1 - Understand Permission Categories

Android groups permissions into categories. Dangerous permissions are the ones to restrict:

Dangerous Permissions (user-facing):
- Location: FINE_LOCATION, COARSE_LOCATION
- Contacts: READ_CONTACTS, WRITE_CONTACTS
- Calendar: READ_CALENDAR, WRITE_CALENDAR
- Camera: CAMERA
- Microphone: RECORD_AUDIO
- SMS: SEND_SMS, READ_SMS
- Phone: READ_PHONE_STATE, CALL_PHONE
- Media/Storage: READ_EXTERNAL_STORAGE, WRITE_EXTERNAL_STORAGE

Normal Permissions (granted automatically, less sensitive):
- Internet access
- Vibration
- Bluetooth

Special Permissions (require Settings access):
- SYSTEM_ALERT_WINDOW (draw over other apps)
- WRITE_SETTINGS (modify system settings)
- INSTALL_UNKNOWN_APPS
- ACCESS_USAGE_STATS (monitor all app usage)

Step 2 - Check App Permissions via Settings

On your Android device:
1. Settings → Apps
2. Select an app
3. Tap "Permissions"
4. View which permissions are granted

This shows what the app *currently* has. But doesn't show what it's *requesting* (permissions in the manifest).

Step 3 - View Full Permission Manifest (ADB Method)

Connect device to computer. In terminal:

```bash
List all packages
adb shell pm list packages

Get permissions for a specific app
adb shell dumpsys package com.example.app | grep -A 20 "granted permissions"

Get requested permissions (in manifest)
adb shell dumpsys package com.example.app | grep -i "permission"
```

Real example (Weather app):

```bash
$ adb shell dumpsys package com.weather.app | grep "permission"
    android.permission.ACCESS_FINE_LOCATION
    android.permission.ACCESS_COARSE_LOCATION
    android.permission.READ_CONTACTS
    android.permission.RECORD_AUDIO
    android.permission.READ_CALENDAR
```

This shows the app requested all these permissions in its manifest, even if you haven't granted them.

Step 4 - Identify Suspicious Permissions

For each app, ask:
- Does this app need location? (weather, maps: yes; note-taking, flashlight: no)
- Does this app need contacts? (social app: maybe; calculator: no)
- Does this app need microphone? (video app: yes; photo app: no)
- Does this app need calendar? (reminder app: yes; game: no)

Red flags:
- Weather app requesting contacts
- Flashlight app requesting location
- Game requesting camera and microphone
- Photo gallery requesting SMS access

Part 2 - Restrict Permissions

Method 1 - Manual Restriction via Settings

On your device:
1. Settings → Apps → [App Name] → Permissions
2. Toggle individual permissions OFF
3. Test app functionality
4. If app works, leave it restricted; if breaks, re-enable

- Chrome: Disable location permission (still works)
- Gmail: Disable camera permission (still works, mail is text)
- Maps: Keep location ON (essential)
- Twitter: Disable location (tweet text doesn't need location)

Method 2 - Bulk Restriction via ADB

```bash
Revoke a specific permission
adb shell pm revoke com.example.app android.permission.ACCESS_FINE_LOCATION

Grant a specific permission
adb shell pm grant com.example.app android.permission.INTERNET

List all permissions for an app (granted)
adb shell pm list permissions -d app com.example.app

Revoke all dangerous permissions for an app
for perm in ACCESS_FINE_LOCATION READ_CONTACTS RECORD_AUDIO READ_CALENDAR; do
    adb shell pm revoke com.example.app android.permission.$perm
done
```

Method 3 - Permission Manager Tools

App-based permission managers:
- Permission Dogs (F-Droid): Shows which apps accessed what, when
- Bouncer ($2): Automatically revoke permissions after app closes
- Exodus Privacy (F-Droid): Shows what trackers are in apps

Device-based management:
- LineageOS Custom ROM: Fine-grained permission control (per-app groups)
- GrapheneOS (Pixel phones): Enhanced permission isolation

Part 3 - Verify Restricted Apps Work

After revoking permissions, test app functionality:

Check logs for crash/errors:
```bash
adb logcat | grep "SecurityException\|PermissionDenied"
```

Test app behavior:
- Weather app: Show forecast (should work without location)
- Gmail: Send email (should work without camera)
- Maps: Search address (may work with coarse location; test without fine location)

Part 4 - Automate Permission Monitoring

Tool 1 - Exodus Privacy Analyzer

Exodus Privacy is a service that scans app APKs and reports:
- What permissions are requested
- What trackers are included (Google Analytics, Firebase, etc.)
- Privacy rating (1-5 stars)

Web version - https://exodus-privacy.eu.org/

Use it to assess apps before installing:
1. Search app name
2. Check permissions listed
3. Check trackers found
4. Read privacy rating

Example results (Facebook):
- Permissions: 13 (many unnecessary)
- Trackers - 8 (Google Analytics, Facebook SDK, Adjust, etc.)
- Privacy: 2/5 stars (poor)

Tool 2 - ADB Automation Script

Create a script to audit all apps:

```bash
#!/bin/bash
audit_permissions.sh

Get all packages
for app in $(adb shell pm list packages | cut -d: -f2); do
    # Get dangerous permissions
    perms=$(adb shell dumpsys package "$app" 2>/dev/null | grep -A 50 "declared permissions" | grep -E "android.permission.(ACCESS|READ|RECORD|CAMERA|CONTACTS|SMS|CALL)" | wc -l)

    if [ "$perms" -gt 3 ]; then
        echo "$app: $perms dangerous permissions"
    fi
done
```

Run:
```bash
chmod +x audit_permissions.sh
./audit_permissions.sh
```

Output:
```
com.facebook.katana: 12 dangerous permissions
com.instagram.android: 10 dangerous permissions
com.twitter.android: 8 dangerous permissions
com.whatsapp: 6 dangerous permissions
```

Tool 3 - Monitor Actual Permission Usage

Use `strace` or `Frida` to see which permissions an app actually uses at runtime:

With Frida (advanced):
```bash
Intercept permission checks
frida -U -f com.example.app -l hook_permissions.js
```

`hook_permissions.js`:
```javascript
Java.perform(function() {
    var Context = Java.use("android.content.Context");
    var ContextCompat = Java.use("androidx.core.content.ContextCompat");

    ContextCompat.checkSelfPermission.implementation = function(context, permission) {
        console.log("[Permission Check] " + permission);
        return 0; // PERMISSION_GRANTED
    };
});
```

This logs every permission check the app makes, showing which permissions it *actually* uses vs. just requests.

Part 5 - Decision Framework

For each app, apply this framework:

```
Does app need [permission]?
 YES → Check if essential to core function
    YES → Grant
    NO → Restrict and test
 NO → Restrict
```

Example decisions:

| App | Permission | Decision | Test |
|-----|-----------|----------|------|
| Weather | Location | GRANT | Shows forecast |
| Weather | Contacts | DENY | Still works |
| Gmail | Microphone | DENY | Can't record, but that's okay |
| Maps | Location | GRANT | Can't navigate without it |
| Spotify | Contacts | DENY | Music plays fine |
| Camera | Camera | GRANT | Can't take photos without it |
| Flashlight | Microphone | DENY | Flashlight turns on |
| Photos | Microphone | DENY | Can view/edit photos |

Part 6 - Automation via Device Policy

For enterprises or advanced users:

MDM (Mobile Device Management)
Use EMM like Intune or MobileIron to deploy permission policies:
```xml
<AppPermissionGrant>
  <AppName>com.example.app</AppName>
  <Permission>android.permission.INTERNET</Permission>
  <Grant>true</Grant>
</AppPermissionGrant>
```

LineageOS Custom ROM
LineageOS offers granular permission control:
- Settings → Privacy → Permissions Manager
- Restrict by-permission across all apps
- Option to spoof location (return fake coordinates)

Work Profile (Android Knox)
Use a work profile on your device:
- Install questionable apps only in work profile
- Work profile has separate permission sandbox
- Isolates personal from work data

Dangerous Permission Categories Explained

Location (FINE_LOCATION, COARSE_LOCATION)
- What it reveals: Home address, work location, frequented places, travel patterns
- Who wants it: Google, Facebook, location-based ads, dating apps
- Mitigation - Grant only to Maps, Weather. Disable otherwise. Use approximate location (coarse) when possible.

Contacts (READ_CONTACTS)
- What it reveals: Social graph, phone numbers, email addresses
- Who wants it: Facebook, LinkedIn, messaging apps
- Mitigation - Grant only to messaging/phone apps. Deny to utilities.

Microphone (RECORD_AUDIO)
- What it reveals: Conversations, background sounds, voice data
- Who wants it: Video apps, voice assistants, call recording
- Mitigation - Grant only to video/call apps. Deny to non-communication apps. Disable when not in use.

Camera (CAMERA)
- What it reveals: Visual data of your environment
- Who wants it: Video apps, video calls, photo apps
- Mitigation - Grant only to intentional camera apps. Check for apps requesting it unnecessarily.

Calendar (READ_CALENDAR)
- What it reveals: Schedule, meetings, travel plans, social commitments
- Who wants it: Google, Microsoft, advertising networks
- Mitigation - Grant only to calendar apps. Deny to utilities and games.

Call Logs (READ_CALL_LOG, CALL_PHONE)
- What it reveals: Who you call, how often, duration of calls
- Who wants it: Social networks, advertising networks, malware
- Mitigation - Grant only to phone app and your carrier. Deny to all others.

SMS (READ_SMS, SEND_SMS)
- What it reveals: Text message content, two-factor authentication codes, banking notifications
- Who wants it: Messaging apps, security software
- Mitigation - Avoid granting to third-party apps. Use native SMS app only. Consider not storing sensitive 2FA via SMS.

Troubleshooting Restricted Apps

If an app crashes after revoking permissions:
1. Check logcat: `adb logcat | grep SecurityException`
2. Grant the permission back
3. Contact app developer (file bug report: "App crashes without [permission]")
4. Consider uninstalling if developer won't fix

If an app behaves strangely (crashes, freezes):
- Re-grant the permission you revoked
- Test again
- If it works, the app requires that permission
- Decision: either grant it or uninstall

If an app requests permission at runtime:
- First time it needs it: dialog appears
- Tap "Deny" to prevent access
- App should gracefully degrade (some features disabled, but app still works)
- If it crashes, see above troubleshooting

Privacy Best Practices Summary

1. Audit before installing: Check permissions via Exodus Privacy
2. Grant minimally: Only what's essential to core function
3. Test restrictively: Revoke and test; re-grant only if broken
4. Monitor constantly: Use Permission Dogs or similar to track actual usage
5. Update regularly: New app versions may request new permissions; review them
6. Use strong alternatives:
 - ProtonMail (email, encrypted)
 - Signal (messaging, encrypted)
 - Mullvad VPN (no logging)
 - Elytra (anonymous profiles)

FAQ

Q: If I revoke a permission, will the app stop working?
A: Maybe. It depends on how the app is coded. Some apps gracefully handle missing permissions; others crash. Test it. If it crashes, the app is poorly designed or genuinely needs that permission.

Q: Is it okay to use "Don't ask again" for all permission denials?
A: Yes. If you deny a permission, you're making a conscious choice. The app will either work without it or fail. That's valuable feedback. Re-enable "ask again" only if you want to reconsider.

Q: Can apps access permissions after I revoke them?
A: No, not on Android 6.0+. Attempting to access a revoked permission throws a SecurityException, which crashes the app if it's not handled properly.

Q: Should I restrict Google's own apps?
A: Google Play Services requests extensive permissions for ads and analytics. Disable permissions that Google's apps don't need for their core function. Google Maps needs location. Gmail doesn't need camera. Be consistent.

Q: Does a VPN protect me from app spying?
A: A VPN encrypts traffic between your phone and the VPN server, preventing your ISP from seeing where you connect. But the app itself can still transmit data (location, contacts, etc.) to its servers. A VPN is a network-level tool; app permissions are the application-level issue. Both matter.

Q: Can I use a ROM like GrapheneOS for better privacy?
A: Yes. GrapheneOS offers isolated permission scopes, making it harder for apps to exploit permissions. But requires a Pixel phone and is complex to set up. For most users, careful permission management on stock Android is sufficient.

Q: How do I audit system apps I can't uninstall?
A: On stock Android, you can't uninstall system apps, but you can disable them:
Settings → Apps → [System App] → Disable (if available)
Or revoke permissions without disabling.
For more control, consider a custom ROM like LineageOS.

Q: Should I be concerned about background services?
A: Yes. Many apps run background services that access permissions while the app isn't open. Use Permission Dogs to see which apps accessed what today. Disable background apps in Battery settings if not essential.

Related Articles

- [How to Audit Android App Permissions: Step-by-Step Guide](/how-to-audit-android-app-permissions-guide/)
- [Android App Permissions Audit Guide 2026](/android-app-permissions-audit-guide-2026/)
- [How To Audit Android App Permissions And Revoke Unnecessary](/how-to-audit-android-app-permissions-and-revoke-unnecessary-/)
- [Audit Android App Permissions with ADB](/android-adb-app-permissions-audit)
- [How to Audit Android App Permissions (2026)](/android-adb-app-permissions-audit/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
