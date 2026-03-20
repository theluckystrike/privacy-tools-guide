---
layout: default
title: "How To Detect And Remove Stalkerware From Android Phone Comp"
description: "A technical guide for detecting and removing stalkerware from Android devices. Includes ADB commands, detection scripts, and removal."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-detect-and-remove-stalkerware-from-android-phone-comp/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Stalkerware represents a serious threat to personal privacy and security. These surveillance applications, often marketed as "employee monitoring" or "child safety" tools, can be secretly installed on Android devices to track location, record calls, capture messages, and access personal data. For developers and power users, understanding how to detect and remove these applications is essential knowledge.

This guide provides technical methods for identifying stalkerware using Android Debug Bridge (ADB), analyzing app behavior, and removing persistent surveillance tools from compromised devices.

## Understanding Stalkerware Behavior

Stalkerware applications typically operate by requesting excessive permissions during installation. Once granted, they run in the background with varying levels of obfuscation. Some common indicators include:

- Applications that request Accessibility Services permission
- Apps that require Device Admin privileges
- Programs that persist after factory reset (indicating firmware-level compromise)
- Unexpected battery drain from unknown processes

Modern stalkerware often hides its icon from the app drawer, making visual detection difficult. The application may appear under generic names like "System Service," "Google Update," or "Device Helper."

## Detection Methods Using ADB

Android Debug Bridge provides the most reliable method for identifying hidden applications. Connect your device via USB with debugging enabled, then execute the following commands.

### Listing All Installed Packages

```bash
adb shell pm list packages -3 -u
```

This command lists all third-party applications installed from unknown sources. The `-u` flag shows uninstalled packages as well, which can reveal recently removed stalkerware.

### Analyzing Application Permissions

```bash
adb shell dumpsys package <package_name> | grep -A 50 "granted=true"
```

Replace `<package_name>` with an application from your list. Look for dangerous permissions such as `READ_CONTACTS`, `READ_SMS`, `ACCESS_FINE_LOCATION`, `RECORD_AUDIO`, or `BIND_ACCESSIBILITY_SERVICE`.

### Checking Accessibility Services

```bash
adb shell settings get secure enabled_accessibility_services
```

Legitimate applications rarely require Accessibility Services. If you find unknown services enabled, investigate immediately.

### Examining Running Processes

```bash
adb shell ps -A | grep -v "system\|android\|com.google"
```

This filters out system processes, leaving only third-party applications currently running on the device.

## Identifying Device Admin Applications

Device Admin access provides deep system privileges that make removal difficult. To identify applications with this status:

```bash
adb shell dumpsys device_policy
```

Review the list of active device administrators. Unknown or suspicious administrators should be revoked immediately.

To remove device admin access programmatically:

```bash
adb shell dpm remove-active-admin --user 0 com.example.suspiciouspackage/.DeviceAdminReceiver
```

## Network Traffic Analysis

Stalkerware must transmit collected data to remote servers. Monitoring network connections can reveal suspicious activity:

```bash
adb shell netstat -tulpn | grep -v "established"
```

Look for connections to unfamiliar IP addresses or domains. You can also use Packet Capture applications (FDroid-hosted) to inspect SSL certificates and destination endpoints.

For developers, integrating network analysis into detection scripts:

```python
import subprocess

def get_connections():
    result = subprocess.run(
        ["adb", "shell", "netstat", "-tan"],
        capture_output=True, text=True
    )
    connections = []
    for line in result.stdout.split('\n'):
        if 'ESTABLISHED' in line:
            parts = line.split()
            if len(parts) >= 4:
                connections.append(parts[3])
    return connections
```

## Removal Strategies

### Standard Uninstall

For applications visible in the app list:

```bash
adb uninstall com.suspicious.package.name
```

### Removing Device Admin Before Uninstall

If the application has device admin status:

1. Navigate to Settings → Security → Device Administrators
2. Disable the suspicious admin
3. Return to Settings → Apps → uninstall

### Removing Bloatware and Persistent Apps

Some stalkerware reinstalls itself or survives factory resets through system app privileges. For advanced removal:

```bash
adb shell pm disable-user --user 0 com.package.name
adb shell pm uninstall --user 0 com.package.name
```

### Factory Reset Considerations

When factory reset is necessary, follow these steps for maximum effectiveness:

1. Enable full device encryption before reset (makes recovery of previous data more difficult)
2. Perform reset from recovery mode (not from settings)
3. After reset, avoid restoring from backups that might include the stalkerware
4. Flash a clean factory image if available for your device

## Prevention and Hardening

### Restricting Installation Sources

Prevent sideloaded stalkerware by restricting app installation:

```bash
adb shell settings put secure install_non_market_apps 0
```

### Monitoring App Operations

Android's built-in Privacy Dashboard (Settings → Privacy) shows which apps accessed sensors, camera, microphone, and location in the past 24 hours. Review this regularly.

### Using Work Profile

Android Work Profile creates a separated environment for sensitive applications. Install potentially problematic apps in the work profile to contain their access:

```bash
adb shell pm create-user --profileOf 0 --managed WorkProfile
```

This isolates applications, limiting their access to personal data.

## Automated Detection Script

For power users managing multiple devices, this Python script provides automated scanning:

```python
import subprocess
import json

SUSPICIOUS_PERMISSIONS = [
    "android.permission.READ_CONTACTS",
    "android.permission.READ_SMS",
    "android.permission.SEND_SMS",
    "android.permission.READ_CALL_LOG",
    "android.permission.RECORD_AUDIO",
    "android.permission.ACCESS_FINE_LOCATION",
    "android.permission.CAMERA"
]

def get_third_party_packages():
    result = subprocess.run(
        ["adb", "shell", "pm", "list", "packages", "-3"],
        capture_output=True, text=True
    )
    packages = [p.replace("package:", "") for p in result.stdout.split()]
    return packages

def check_permissions(package):
    result = subprocess.run(
        ["adb", "shell", "dumpsys", "package", package],
        capture_output=True, text=True
    )
    granted = []
    for perm in SUSPICIOUS_PERMISSIONS:
        if perm in result.stdout and "granted=true" in result.stdout:
            granted.append(perm)
    return granted

def scan_device():
    print("Scanning device for stalkerware indicators...")
    packages = get_third_party_packages()
    findings = []
    
    for pkg in packages:
        perms = check_permissions(pkg)
        if len(perms) > 2:
            findings.append({"package": pkg, "permissions": perms})
    
    if findings:
        print(f"Found {len(findings)} suspicious applications:")
        for f in findings:
            print(f"  - {f['package']}: {', '.join(f['permissions'])}")
    else:
        print("No suspicious applications detected.")
    
    return findings

if __name__ == "__main__":
    scan_device()
```

Run this script periodically to audit devices for suspicious permission patterns.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
