---
layout: default
title: "How to Detect Hidden Trackers in Android"
description: "Use ADB, TrackerControl, Exodus Privacy, and network analysis to find which Android apps are sending your data to ad networks, analytics firms, and data brokers"
date: 2026-03-21
author: theluckystrike
permalink: /detect-hidden-trackers-android-apps/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Android apps regularly include third-party tracking SDKs — libraries built by advertising companies, analytics firms, and data brokers that are embedded into apps you install. The app developer may not have written any surveillance code themselves; they just included a popular analytics library that phones home to 15 different servers. This guide shows you how to find those trackers.

## Method 1: Exodus Privacy — Static Analysis

Exodus Privacy (exodus-privacy.eu.org) maintains a database of known Android tracking SDKs. Their scanner analyzes APK files without running them and identifies embedded trackers by matching code signatures.

### Web interface

1. Go to reports.exodus-privacy.eu.org
2. Search for any app by package name (e.g., `com.instagram.android`)
3. The report shows:
   - List of identified trackers with descriptions
   - All permissions requested by the app
   - Network endpoints the app can contact

This is the fastest way to check a specific app before installing it.

### Command-line with exodus-standalone

```bash
# Install
pip3 install exodus-standalone

# Download an APK (use APKPure, APKMirror, or extract from device)
# Or extract from a device already running the app:
adb shell pm path com.example.app
# Returns: package:/data/app/com.example.app-1/base.apk

adb pull /data/app/com.example.app-1/base.apk

# Analyze
exodus-standalone -i base.apk

# Output shows trackers, permissions, and URLs
```

## Method 2: TrackerControl — Runtime Analysis

TrackerControl is an Android app that monitors all network traffic from every installed app in real time, without rooting your device. It works by creating a local VPN that routes all traffic through itself for inspection.

Install from F-Droid (strongly preferred over Play Store for this privacy tool):
https://f-droid.org/packages/net.kollnig.missioncontrol.fdroid/

Once running:
- Open TrackerControl → Apps tab → select any app → tap to see active tracking domains
- Sort by "Trackers" to see which apps have the most tracker connections
- Tap any tracker to see which company owns it and what data they collect

TrackerControl blocks trackers at the network level — you can toggle blocking per app or per tracker.

### What TrackerControl reveals

After running for 24-48 hours, check the "Activity" tab. You'll typically find:

- **Google Analytics/Firebase** — nearly universal; collects usage events
- **Facebook SDK** — sends device fingerprint and app events to Facebook even if you're not logged in
- **Adjust, AppsFlyer, Kochava** — install attribution trackers that link your device identity across apps
- **Braze** — push notification and analytics platform with persistent device IDs
- **Crashlytics** — crash reporting (lower risk, but still exfiltrates data)

## Method 3: ADB Permission Audit

ADB (Android Debug Bridge) lets you inspect what permissions each app has been granted, including permissions the app holds but doesn't display in its own settings:

```bash
# Enable USB debugging on your Android device:
# Settings → Developer Options → USB Debugging

# Connect via USB and verify connection
adb devices

# List all packages with their permissions
adb shell pm list permissions -g -f > all_permissions.txt

# Check permissions for a specific package
adb shell dumpsys package com.example.app | grep "permission"

# Find apps with dangerous permissions
adb shell pm list packages -f > packages.txt

# Check which apps have location permission
adb shell pm list permissions -d | grep -i location
```

More targeted — find all apps with access to location, contacts, or microphone:

```bash
# Location-granted apps
adb shell appops query-op android:coarse_location allow
adb shell appops query-op android:fine_location allow

# Microphone access
adb shell appops query-op android:record_audio allow

# Background location (apps that can track you when not in use)
adb shell dumpsys location | grep "background"
```

## Method 4: Network Traffic Capture

The most thorough method is capturing network traffic from the phone itself. This shows you actual data being transmitted, not just potential capability.

### Using netcapture / PCAPdroid (no root required)

Install PCAPdroid from F-Droid:
https://f-droid.org/packages/com.emanuelef.remote_capture/

PCAPdroid captures network traffic using the same local VPN technique as TrackerControl. After capturing, you can:
- Export as PCAP and analyze in Wireshark on your PC
- View connections in-app with app-to-destination mapping
- See payload sizes and transmission frequency

```bash
# Transfer PCAP from phone to PC
adb pull /storage/emulated/0/PCAPdroid/captures/capture.pcap .

# Analyze in tshark
tshark -r capture.pcap -T fields -e ip.dst -e dns.qry.name | sort -u
```

### Analyzing the captured traffic

Look for:
- Connections to known tracking domains (compare against disconnect.me's tracking protection list)
- Large POST requests immediately after app launch (initial device fingerprint submission)
- Periodic background connections (beacon pings every few minutes)
- Connections to unusual TLDs or random-looking subdomains (may indicate obfuscated tracking)

```bash
# Download disconnect.me tracker list
curl -O https://raw.githubusercontent.com/nicktacular/dnscrypt-disconnect-me-trackers/master/domains.txt

# Cross-reference captured domains
tshark -r capture.pcap -T fields -e dns.qry.name | sort -u > captured_domains.txt
comm -12 <(sort domains.txt) <(sort captured_domains.txt)
```

## Method 5: ClassyShark3xodus — On-Device Static Analysis

ClassyShark3xodus is an Android app that scans APKs installed on your device and reports embedded trackers, similar to Exodus but running locally:

Available on F-Droid: https://f-droid.org/packages/com.oF2pks.classyshark3xodus/

Launch it, point it at any installed app, and it generates a report of found trackers using the Exodus database — all without sending the APK to a remote server.

## Interpreting Results: What to Do About Trackers

Not all trackers are equally bad. A rough hierarchy:

**High concern:**
- Facebook SDK in an app you don't associate with your Facebook identity
- Location trackers in apps with no legitimate location need
- Advertising ID collectors in apps marketed as "privacy tools"

**Medium concern:**
- General analytics (Firebase, Amplitude) — collects usage patterns, rarely identifiable PII
- Crash reporters (Crashlytics, Sentry) — useful to developers, sends device info on crashes

**Lower concern:**
- Performance monitoring that stays within the app's own domain

**Actions to take:**

```bash
# Reset your Advertising ID (limits cross-app tracking correlation)
# Settings → Google → Ads → Reset advertising ID

# For Android 12+: opt out entirely
# Settings → Privacy → Ads → Delete advertising ID

# Revoke permissions for apps that don't need them
adb shell pm revoke com.example.app android.permission.ACCESS_FINE_LOCATION

# For apps you can't uninstall (system apps), disable them
adb shell pm disable-user --user 0 com.google.android.gms.ads
```

## Related Reading

- [Android ADB Commands for Removing Bloatware That Tracks Users](/android-adb-commands-for-removing-bloatware-that-tracks-user/)
- [Android App Permissions Audit Guide 2026](/android-app-permissions-audit-guide-2026/)
- [Android Privacy Best Practices 2026](/android-privacy-best-practices-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
