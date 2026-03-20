---

layout: default
title: "CalyxOS Datura Firewall Setup: Controlling Per-App Internet Access Without Root"
description: "Learn how to configure the Datura firewall in CalyxOS to control per-app internet access without root. Practical examples for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /calyxos-datura-firewall-setup-controlling-per-app-internet-a/
categories: [security, guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

CalyxOS includes a powerful built-in firewall called Datura that lets you control which apps can access the internet. Unlike traditional Android firewall solutions that require root access, Datura operates at the system level through CalyxOS's microG integration. This makes it accessible to anyone running CalyxOS on supported devices like Pixel phones.

## What is Datura Firewall?

Datura is a netfilter-based firewall implementation designed specifically for CalyxOS. It leverages Linux's native packet filtering capabilities to block or allow network traffic on a per-application basis. The firewall operates at the iptables level, meaning it can filter both IPv4 and IPv6 traffic without requiring special permissions or root access.

The key advantage of Datura over third-party firewall apps is its integration with the operating system. It runs as a privileged system service, giving it more control over network packets than user-space firewall applications that rely on VPN-based routing.

## Accessing Datura Firewall Settings

To access the firewall configuration, navigate to Settings on your CalyxOS device, then go to Network & Internet, and finally select Datura Firewall. Alternatively, you can find it under Settings > Privacy > Firewall.

The interface presents three main tabs:
- **All Apps**: A complete list of installed applications
- **System Apps**: Pre-installed system applications
- **User Apps**: Applications you've installed

Each app displays its icon, name, and current network permissions with visual indicators showing whether it has WiFi, mobile data, or VPN access.

## Basic Firewall Configuration

The Datura firewall operates on a whitelist model by default, meaning all traffic is allowed unless you explicitly block an app. This approach ensures that new apps work immediately after installation while giving you granular control to restrict their network behavior.

To block an app's internet access:

1. Open the Datura Firewall settings
2. Navigate to the appropriate app category (All Apps, System Apps, or User Apps)
3. Find the application you want to restrict
4. Toggle off the permissions you want to revoke—WiFi, Mobile Data, or both

For example, to completely block a pre-installed bloatware application from accessing the internet:

```bash
# Check current firewall status via adb (if needed for debugging)
adb shell dumpsys firewall
```

This command outputs detailed information about current firewall rules, including which apps are allowed or blocked on different network interfaces.

## Practical Use Cases for Power Users

### Blocking Analytics and Telemetry

Many applications include analytics SDKs that transmit usage data to remote servers. Using Datura, you can block these network connections without uninstalling the applications. This is particularly useful for:

- Pre-installed system apps that cannot be uninstalled
- Apps you need for specific functionality but want to minimize data collection
- Applications that crash frequently due to network-related issues

Common candidates for blocking include system update services, carrier services, and manufacturer bloatware that collects telemetry data.

### Creating Network Profiles

While Datura doesn't have native profile support, you can achieve similar functionality by manually adjusting rules based on your needs. Consider creating a systematic approach:

```bash
# Example workflow for managing app permissions
# 1. List all apps with network access
adb shell pm list packages -3 | while read pkg; do
    echo "Package: $pkg"
done
```

For mobile-only access scenarios, you might allow certain apps on mobile data while blocking WiFi access. This is useful for applications that behave differently depending on network type.

### Isolating Sensitive Applications

For applications that handle sensitive data—banking apps, password managers, cryptocurrency wallets—you might want to restrict their network access entirely or only allow connections through specific VPN configurations.

To review which apps have unrestricted network access:

1. Open Datura Firewall
2. Tap the filter icon in the top-right corner
3. Select "Show apps with WiFi access" or "Show apps with mobile data access"

This helps identify apps that may need stricter controls.

## Advanced Configuration via ADB

For developers who want more control, Datura supports command-line configuration through ADB (Android Debug Bridge). This enables scripting and automation of firewall rules.

### Querying Current Rules

```bash
# Get detailed firewall configuration for a specific package
adb shell cmd firewall enable <package_name>
adb shell cmd firewall disable <package_name>
```

These commands allow you to toggle firewall rules for specific applications directly from your development machine.

### Bulk Operations

You can create scripts to manage multiple applications at once:

```bash
#!/bin/bash
# Block network access for multiple packages

PACKAGES=(
    "com.example.bloatware1"
    "com.example.bloatware2"
    "com.example.tracker"
)

for pkg in "${PACKAGES[@]}"; do
    adb shell cmd firewall set-uid-rule $pkg both BLOCK
    echo "Blocked $pkg"
done
```

This script demonstrates how to programmatically apply consistent firewall policies across multiple applications.

## Understanding Network Permission Indicators

Datura uses a clear visual system to indicate network permissions:

- **Green checkmark**: App has unrestricted network access
- **Orange dot**: App has limited access (specific networks only)
- **Red X**: App is completely blocked

Understanding these indicators helps you quickly audit your device's network security posture.

## Troubleshooting Connectivity Issues

If an application fails to connect after configuring firewall rules, the issue is often a blocked dependency. Many apps rely on backend services, CDN resources, or third-party APIs that might not be obvious.

To diagnose:

1. Check if the app works with all firewall rules disabled
2. Enable rules one category at a time (WiFi vs. Mobile Data)
3. Review app documentation for required domains
4. Temporarily allow the app to confirm the issue is network-related

Some applications require specific domain access for push notifications, license verification, or content delivery. Blocking these can cause functionality issues beyond simple connectivity failures.

## Security Considerations

The Datura firewall significantly enhances your privacy posture by giving you control over data exfiltration. However, keep in mind:

- Firewall rules persist across app updates
- Some apps may behave unexpectedly when network access is restricted
- System updates may reset or modify firewall configurations
- VPN-based firewall apps cannot coexist with Datura's system-level filtering

Regularly audit your firewall rules to ensure they align with your current privacy requirements.

## Getting Started

Begin by reviewing all applications with network access in your Datura Firewall settings. Identify apps that don't need internet connectivity and restrict them. For developers and power users, explore the ADB interface for bulk operations and automation.

The Datura firewall provides enterprise-grade network filtering without requiring root access, making CalyxOS an excellent choice for privacy-conscious users who need granular control over their device's network behavior.

## Related Reading

- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [How to Configure Per App VPN on Android Without Root.](/privacy-tools-guide/how-to-configure-per-app-vpn-on-android-without-root/)
- [CalyxOS MicroG Setup Guide: Getting Google Apps Working.](/privacy-tools-guide/calyxos-microg-setup-guide-getting-google-apps-working-without-google-services/)
- [Secure VoIP Setup for Private Phone Calls Without.](/privacy-tools-guide/secure-voip-setup-for-private-phone-calls-without-carrier-in/)

Built by