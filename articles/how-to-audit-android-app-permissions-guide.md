---
layout: default
title: "How to Audit Android App Permissions: Step-by-Step Guide"
description: "Technical guide to auditing Android app permissions using ADB, permission groups, automation scripts, and recommended permission configurations"
date: 2026-03-20
last_modified_at: 2026-03-20
author: "Privacy Tools Guide"
permalink: /how-to-audit-android-app-permissions-guide/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of]
reviewed: true
score: 8
voice-checked: true
intent-checked: true---
---
layout: default
title: "How to Audit Android App Permissions: Step-by-Step Guide"
description: "Technical guide to auditing Android app permissions using ADB, permission groups, automation scripts, and recommended permission configurations"
date: 2026-03-20
last_modified_at: 2026-03-20
author: "Privacy Tools Guide"
permalink: /how-to-audit-android-app-permissions-guide/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of]
reviewed: true
score: 8
voice-checked: true
intent-checked: true---

{% raw %}

Android apps request permissions to access sensitive data: camera, microphone, location, contacts, photos. Most users grant all permissions during installation without understanding what access they grant. Auditing app permissions reveals which apps have unnecessary access, which permissions are actually used, and whether you can safely restrict access. This guide provides step-by-step techniques for permission audits.

## Key Takeaways

- **Most users grant all**: permissions during installation without understanding what access they grant.
- **Auditing app permissions reveals**: which apps have unnecessary access, which permissions are actually used, and whether you can safely restrict access.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Consider a security review**: if your application handles sensitive user data.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Android Permission Architecture

Android permissions operate through permission groups. Apps declare required permissions in their manifest; Android groups related permissions together. When an app requests camera access, you can grant or deny it. Unlike older Android versions, you can revoke permissions after installation without uninstalling the app.

**Key permission groups**:

- **Location**: GPS location, coarse location (based on WiFi/cellular)
- **Camera**: Takes photos/videos
- **Microphone**: Records audio
- **Contacts**: Reads contact list
- **Calendar**: Reads calendar events
- **Call logs**: Reads call history
- **SMS**: Reads text messages
- **Files**: Reads photos, videos, documents
- **Phone**: Reads phone number, SIM details

Each group contains related permissions. Granting "Camera" permission grants both CAMERA and associated image recording permissions.

### Step 2: Method 1: Using Android Settings (Manual)

The graphical approach works for systematic audits of all installed apps.

**Step 1: Access Permission Manager**

```
Settings → Privacy and safety → Permissions → Permission Manager
```

This shows all permission groups. On some Android versions:

```
Settings → Apps → Permissions → [Permission name]
```

**Step 2: Review Each Permission Group**

For each permission (Location, Camera, Microphone, Contacts, SMS):

1. Tap the permission name
2. See which apps have access
3. Evaluate if each app actually needs that permission

**Real-world example: Location permissions**

```
Settings → Privacy → Permission Manager → Location

Apps with location access:
- Google Maps: ✓ Legitimate (maps requires location)
- Weather app: ✓ Legitimate (shows local weather)
- Instagram: ✗ Why? Instagram doesn't need precise location
- Spotify: ✗ Why? Music app requests location
- Banking app: ? Maybe (fraud detection? geolocation for security)

Action:
- Click Instagram → Change to "Don't allow"
- Click Spotify → Change to "Don't allow"
- Verify banking app actually uses location: contact support if unclear
```

**Step 3: Deny Unnecessary Permissions**

For apps that don't need a permission, change from "Allow" to "Don't allow" or "Allow only while using the app."

Android permission levels:

```
Allow all the time (always): App can access even when you're not using it
Allow only while using the app: App loses access when you switch apps
Ask every time: System prompts each access attempt
Don't allow: Complete denial
```

For most apps, "Allow only while using the app" is the safest middle ground.

### Step 3: Method 2: Using ADB (Android Debug Bridge)

ADB provides command-line access to detailed permission information and control.

**Installation**:

```bash
# macOS with Homebrew
brew install android-platform-tools

# Linux
sudo apt install android-sdk-platform-tools

# Windows
Download from developer.android.com
```

**Enable Developer Mode on phone**:

1. Settings → About Phone
2. Tap "Build Number" 7 times
3. Go back, find "Developer options"
4. Enable "USB Debugging"

**Connect phone via USB**:

```bash
adb devices
# Should show your device with status "device"
```

**Inspect all installed apps and permissions**:

```bash
# List all installed packages
adb shell pm list packages

# Show detailed info for a specific app (e.g., Instagram)
adb shell dumpsys package com.instagram.android

# Shows:
# - Declared permissions (what app requests)
# - Granted permissions (what you've allowed)
# - Installation date
# - Version info
```

**List all permissions and their status**:

```bash
adb shell pm list permissions -d

# Output:
# android.permission.CAMERA: protection level=dangerous
# android.permission.LOCATION_HARDWARE: protection level=dangerous
# (and many more)
```

**Check which apps have a specific permission**:

```bash
# Which apps have location permission?
adb shell cmd appops query-ops | grep COARSE_LOCATION

# Check camera permissions
adb shell dumpsys package | grep "Camera"
```

**Revoke permissions via ADB**:

```bash
# Revoke Instagram's location permission
adb shell pm revoke com.instagram.android android.permission.ACCESS_FINE_LOCATION

# Grant a permission back
adb shell pm grant com.instagram.android android.permission.CAMERA

# Verify the change
adb shell dumpsys package com.instagram.android | grep "FINE_LOCATION"
```

### Step 4: Method 3: Permission Audit Script

Automate permission auditing with a shell script:

```bash
#!/bin/bash
# Android Permission Audit Script

echo "=== Android App Permission Audit ==="
echo ""

# Get all installed packages
packages=$(adb shell pm list packages -3 | cut -d: -f2)

# Dangerous permissions to check
dangerous_perms=(
  "android.permission.ACCESS_FINE_LOCATION"
  "android.permission.ACCESS_COARSE_LOCATION"
  "android.permission.CAMERA"
  "android.permission.RECORD_AUDIO"
  "android.permission.READ_CONTACTS"
  "android.permission.READ_CALENDAR"
  "android.permission.READ_CALL_LOG"
  "android.permission.READ_SMS"
  "android.permission.READ_PHONE_STATE"
)

# For each app, check dangerous permissions
for package in $packages; do
  has_dangerous=false

  for perm in "${dangerous_perms[@]}"; do
    # Check if app requests this permission
    result=$(adb shell dumpsys package "$package" 2>/dev/null | grep "$perm" | head -1)

    if [[ ! -z "$result" ]]; then
      if [[ $has_dangerous == false ]]; then
        echo "=== $package ==="
        has_dangerous=true
      fi
      echo "  - $perm"
    fi
  done

  if [[ $has_dangerous == true ]]; then
    echo ""
  fi
done

echo "=== Audit Complete ==="
echo "Review apps above and revoke unnecessary permissions:"
echo "adb shell pm revoke <package> <permission>"
```

Run the script:

```bash
chmod +x permission-audit.sh
./permission-audit.sh > permissions_report.txt

# Review report
cat permissions_report.txt
```

### Step 5: Method 4: Using Exodus Privacy

Exodus Privacy is a web tool that analyzes APK files and shows which permissions apps request and why.

**How it works**:

1. Go to exodus-privacy.eu.org
2. Search for an app (e.g., "Instagram")
3. See all permissions requested
4. See trackers embedded in the app
5. See which SDKs (software libraries) request permissions

**Example analysis for Spotify**:

```
App: Spotify
Version: Latest

Permissions:
- ACCESS_FINE_LOCATION: Music recommendations (claimed by Spotify)
- RECORD_AUDIO: Playing music through microphone (unnecessary)
- READ_CONTACTS: Sharing playlists with contacts (legitimate)

Trackers: 12 identified
- Google Analytics
- Facebook SDK
- Adjust (analytics)
- etc.

Recommendation: Revoke RECORD_AUDIO, monitor trackers
```

### Step 6: Recommended Permission Strategy by App Type

**Social media apps** (Instagram, TikTok, Snapchat):
- Camera: Allow while using app ✓
- Microphone: Allow while using app ✓
- Contacts: Don't allow ✗ (unless you specifically use "find friends" feature)
- Location: Don't allow ✗
- Photos: Allow while using app ✓
- Calendar: Don't allow ✗

**Maps and navigation** (Google Maps, Apple Maps):
- Location: Allow all the time ✓ (navigation needs background access)
- Camera: Don't allow ✗
- Microphone: Allow while using app (for voice commands) ✓

**Banking apps**:
- Camera: Allow while using app (mobile check deposits) ✓
- Microphone: Don't allow ✗
- Location: Don't allow ✗ (unless explicitly needed)
- Photos: Allow while using app ✓
- Contacts: Don't allow ✗

**Messaging apps** (WhatsApp, Signal, Telegram):
- Camera: Allow while using app ✓
- Microphone: Allow while using app ✓
- Contacts: Allow ✓ (app requires this to work)
- Location: Don't allow ✗
- Files: Allow while using app ✓

**Email apps**:
- Camera: Don't allow ✗
- Microphone: Don't allow ✗
- Contacts: Allow ✓ (for address book integration)
- Files: Allow ✓ (for attachments)
- Location: Don't allow ✗

**Fitness apps** (Strava, MyFitnessPal):
- Location: Allow all the time ✓ (tracking workouts)
- Camera: Don't allow ✗
- Health/fitness data: Allow ✓
- Contacts: Don't allow ✗

### Step 7: Detecting Permission Abuse

Apps sometimes request suspicious permissions. Signs of abuse:

**High-permission apps that don't need them**:
- Flashlight app requesting camera: Legitimate (provides light)
- Flashlight app requesting contacts: Red flag (no reason for a flashlight)
- Keyboard requesting location: Red flag (keyboards don't need location)

**Verify in Exodus Privacy**:
1. Search for the app
2. Review claimed reasons for permissions
3. If reasons seem off, the app might be spyware

**Example red flags**:
- App claiming location is "for better ads" (suspicious)
- App claiming to need contacts for "improving search" (suspicious)
- App using multiple trackers beyond necessity

### Step 8: Automated Permission Monitoring

Enable Google Play Protect:

```
Settings → Security → Google Play Protect → Enable
```

Google Play Protect automatically scans apps for malicious behavior and permission abuse. It won't catch all issues but provides baseline protection.

### Step 9: Complete Permission Audit Checklist

[ ] Install ADB and connect phone
[ ] Run permission audit script or manual review
[ ] For each app, verify necessity of permissions
[ ] For suspicious apps, check Exodus Privacy analysis
[ ] Revoke unnecessary permissions via ADB or settings
[ ] Document any permission denials that break app functionality
[ ] Set up Google Play Protect monitoring
[ ] Repeat audit quarterly

## Troubleshooting

**App stops working after revoking permission**: The app might require that permission even if it shouldn't. Options:
1. Grant permission back and accept the privacy trade-off
2. Find an alternative app that doesn't require that permission
3. Contact app developer to request the feature work without permission

**ADB device not found**:
- Ensure USB Debugging is enabled
- Try different USB cable
- Restart adb server: `adb kill-server && adb start-server`

**Permission revocation doesn't stick**:
- Some apps may request permission again
- Some system apps require permissions by default
- For problematic apps, consider uninstalling or using privacy-focused alternative

## Frequently Asked Questions

**How long does it take to audit android app permissions: step-by-step guide?**

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

- [Audit Android App Permissions with ADB](/privacy-tools-guide/android-adb-app-permissions-audit)
- [Android App Permissions Audit Guide 2026](/privacy-tools-guide/android-app-permissions-audit-guide-2026/)
- [How to Audit Android App Permissions (2026)](/privacy-tools-guide/android-adb-app-permissions-audit/)
- [How To Audit Android App Permissions And Revoke Unnecessary](/privacy-tools-guide/how-to-audit-android-app-permissions-and-revoke-unnecessary-/)
- [Android Storage Scopes How Modern Permissions Limit App Acce](/privacy-tools-guide/android-storage-scopes-how-modern-permissions-limit-app-acce/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
