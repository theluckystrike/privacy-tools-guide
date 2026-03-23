---
layout: default
title: "CalyxOS Datura Firewall Setup: Controlling Per-App"
description: "Learn how to configure the Datura firewall in CalyxOS to control per-app internet access without root. Practical examples for developers and power users"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /calyxos-datura-firewall-setup-controlling-per-app-internet-a/
categories: [security, guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

CalyxOS includes a powerful built-in firewall called Datura that lets you control which apps can access the internet. Unlike traditional Android firewall solutions that require root access, Datura operates at the system level through CalyxOS's microG integration. This makes it accessible to anyone running CalyxOS on supported devices like Pixel phones.

Table of Contents

- [What is Datura Firewall?](#what-is-datura-firewall)
- [Prerequisites](#prerequisites)
- [Advanced Configuration via ADB](#advanced-configuration-via-adb)
- [Troubleshooting Connectivity Issues](#troubleshooting-connectivity-issues)
- [Security Considerations](#security-considerations)
- [Getting Started](#getting-started)
- [Combining Datura with Android VPN Services](#combining-datura-with-android-vpn-services)
- [Performance Impact Analysis](#performance-impact-analysis)
- [Troubleshooting Datura Connectivity Issues](#troubleshooting-datura-connectivity-issues)

What is Datura Firewall?

Datura is a netfilter-based firewall implementation designed specifically for CalyxOS. It uses Linux's native packet filtering capabilities to block or allow network traffic on a per-application basis. The firewall operates at the iptables level, meaning it can filter both IPv4 and IPv6 traffic without requiring special permissions or root access.

The key advantage of Datura over third-party firewall apps is its integration with the operating system. It runs as a privileged system service, giving it more control over network packets than user-space firewall applications that rely on VPN-based routing.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1: Access Datura Firewall Settings

To access the firewall configuration, navigate to Settings on your CalyxOS device, then go to Network & Internet, and finally select Datura Firewall. Alternatively, you can find it under Settings > Privacy > Firewall.

The interface presents three main tabs:
- All Apps: A complete list of installed applications
- System Apps: Pre-installed system applications
- User Apps: Applications you've installed

Each app displays its icon, name, and current network permissions with visual indicators showing whether it has WiFi, mobile data, or VPN access.

Step 2: Basic Firewall Configuration

The Datura firewall operates on a whitelist model by default, meaning all traffic is allowed unless you explicitly block an app. This approach ensures that new apps work immediately after installation while giving you granular control to restrict their network behavior.

To block an app's internet access:

1. Open the Datura Firewall settings
2. Navigate to the appropriate app category (All Apps, System Apps, or User Apps)
3. Find the application you want to restrict
4. Toggle off the permissions you want to revoke, WiFi, Mobile Data, or both

For example, to completely block a pre-installed bloatware application from accessing the internet:

```bash
Check current firewall status via adb (if needed for debugging)
adb shell dumpsys firewall
```

This command outputs detailed information about current firewall rules, including which apps are allowed or blocked on different network interfaces.

Step 3: Practical Use Cases for Power Users

Blocking Analytics and Telemetry

Many applications include analytics SDKs that transmit usage data to remote servers. Using Datura, you can block these network connections without uninstalling the applications. This is particularly useful for:

- Pre-installed system apps that cannot be uninstalled
- Apps you need for specific functionality but want to minimize data collection
- Applications that crash frequently due to network-related issues

Common candidates for blocking include system update services, carrier services, and manufacturer bloatware that collects telemetry data.

Creating Network Profiles

While Datura doesn't have native profile support, you can achieve similar functionality by manually adjusting rules based on your needs. Consider creating a systematic approach:

```bash
Example workflow for managing app permissions
1. List all apps with network access
adb shell pm list packages -3 | while read pkg; do
    echo "Package: $pkg"
done
```

For mobile-only access scenarios, you might allow certain apps on mobile data while blocking WiFi access. This is useful for applications that behave differently depending on network type.

Isolating Sensitive Applications

For applications that handle sensitive data, banking apps, password managers, cryptocurrency wallets, you might want to restrict their network access entirely or only allow connections through specific VPN configurations.

To review which apps have unrestricted network access:

1. Open Datura Firewall
2. Tap the filter icon in the top-right corner
3. Select "Show apps with WiFi access" or "Show apps with mobile data access"

This helps identify apps that may need stricter controls.

Advanced Configuration via ADB

For developers who want more control, Datura supports command-line configuration through ADB (Android Debug Bridge). This enables scripting and automation of firewall rules.

Querying Current Rules

```bash
Get detailed firewall configuration for a specific package
adb shell cmd firewall enable <package_name>
adb shell cmd firewall disable <package_name>
```

These commands allow you to toggle firewall rules for specific applications directly from your development machine.

Bulk Operations

You can create scripts to manage multiple applications at once:

```bash
#!/bin/bash
Block network access for multiple packages

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

Step 4: Understand Network Permission Indicators

Datura uses a clear visual system to indicate network permissions:

- Green checkmark: App has unrestricted network access
- Orange dot: App has limited access (specific networks only)
- Red X: App is completely blocked

Understanding these indicators helps you quickly audit your device's network security posture.

Troubleshooting Connectivity Issues

If an application fails to connect after configuring firewall rules, the issue is often a blocked dependency. Many apps rely on backend services, CDN resources, or third-party APIs that might not be obvious.

To diagnose:

1. Check if the app works with all firewall rules disabled
2. Enable rules one category at a time (WiFi vs. Mobile Data)
3. Review app documentation for required domains
4. Temporarily allow the app to confirm the issue is network-related

Some applications require specific domain access for push notifications, license verification, or content delivery. Blocking these can cause functionality issues beyond simple connectivity failures.

Security Considerations

The Datura firewall significantly enhances your privacy posture by giving you control over data exfiltration. However, keep in mind:

- Firewall rules persist across app updates
- Some apps may behave unexpectedly when network access is restricted
- System updates may reset or modify firewall configurations
- VPN-based firewall apps cannot coexist with Datura's system-level filtering

Regularly audit your firewall rules to ensure they align with your current privacy requirements.

Getting Started

Begin by reviewing all applications with network access in your Datura Firewall settings. Identify apps that don't need internet connectivity and restrict them. For developers and power users, explore the ADB interface for bulk operations and automation.

The Datura firewall provides enterprise-grade network filtering without requiring root access, making CalyxOS an excellent choice for privacy-conscious users who need granular control over their device's network behavior.

Step 5: Deep Dive: Netfilter Architecture on Android

Datura uses Linux netfilter (kernel-level packet filtering) adapted for Android. Understanding the architecture explains its power and limitations.

Traditional user-space VPN firewalls (like NetGuard or AFwall+) use the VPN interface to intercept traffic. This means the VPN slot is occupied, you cannot use a real VPN simultaneously. Datura operates at iptables level, below the VPN layer:

```
Application
     ↓
Network Stack
     ↓
netfilter (Datura)  ← Filtering happens here
     ↓
iptables rules
     ↓
VPN/Network interface
```

This layering allows Datura to coexist with other VPN apps. The trade-off: Datura cannot block traffic at the individual app level if a compromised or malicious app tunnels data through legitimate proxies.

Step 6: Examining iptables Rules on CalyxOS

Power users can inspect actual filtering rules from the kernel:

```bash
Connect via ADB
adb shell

List current iptables rules (requires appropriate permissions)
su
iptables -L -n -v

Filter rules by chain
iptables -L INPUT -n
iptables -L OUTPUT -n

Check for per-app rules
iptables -L OUTPUT -n | grep -i "com.example"
```

The output shows how Datura implements rules. For each blocked app, you typically see DROP rules that discard packets matching the app's UID.

Step 7: DNS Filtering Within Datura

Datura can filter DNS traffic separately from data traffic. If you want to block ads network-wide, configuring Datura to block DNS to ad servers is one approach:

```bash
View current DNS rules
adb shell su -c "iptables -t nat -L PREROUTING -n | grep 'dns'"

Common ad server DNS: 216.58.x.x (Google ads), 44.55.82.x (DoubleClick), etc.
Blocking DNS prevents apps from resolving these domains
```

However, modern ad networks use domain shadowing and CDNs, making DNS-level blocking incomplete. App-level blocking through Datura is more effective.

Combining Datura with Android VPN Services

Unlike VPN-based firewalls, Datura allows a real VPN connection simultaneously:

```
Setup:
1. Configure VPN for encrypted tunnel (ProtonVPN, Mullvad, etc.)
2. Configure Datura to block apps from accessing unencrypted network
3. Result: Apps blocked at app-level by Datura + encrypted via VPN
```

This hybrid approach provides defense in depth. Datura acts as an additional filter, preventing accidental data leakage if VPN drops.

Step 8: Detecting Datura-Blocked Traffic

Apps sometimes retry blocked connections repeatedly, draining battery. Monitor this:

```bash
Check for repeated connection attempts
adb shell dumpsys connectivity

High error rates or repeated "connection refused" indicate blocked apps
This is expected but useful to identify misconfigured permissions
```

For developers testing their apps, ensure your app handles network failures gracefully:

```kotlin
// Graceful degradation when network is blocked
try {
    val response = httpClient.get(url)
} catch (e: IOException) {
    // Network blocked or unavailable
    showOfflineMessage()
    return@launch
}
```

Step 9: Build Custom Firewall Policies with ADB

Create scripts to apply consistent policies across devices:

```bash
#!/bin/bash
apply_firewall_policy.sh
Policy: Block all non-essential apps from mobile data, allow only WiFi

DEVICE=$1
ADB_CMD="adb -s $DEVICE shell"

List of apps to restrict to WiFi only
WIFI_ONLY_APPS=(
    "com.instagram.android"
    "com.facebook.katana"
    "com.twitter.android"
    "com.reddit.frontpage"
)

for app in "${WIFI_ONLY_APPS[@]}"; do
    # Check if app exists
    if $ADB_CMD pm list packages | grep -q "$app"; then
        $ADB_CMD cmd firewall set-uid-rule "$app" mobile BLOCK
        echo "Blocked $app from mobile data"
    fi
done
```

Step 10: IPv6 Filtering Considerations

Modern networks use both IPv4 and IPv6. Datura filters both, but misconfiguration is common:

```bash
Check if IPv6 is enabled
adb shell ip -6 route show

Verify Datura blocks IPv6 for restricted apps
adb shell su -c "ip6tables -L OUTPUT -n | grep DROP"
```

If IPv6 is enabled but Datura only filters IPv4, apps can leak traffic through IPv6 tunnels. Ensure your CalyxOS version filters both protocols.

For maximum privacy on networks supporting IPv6, disable IPv6 entirely:

```bash
adb shell su -c "sysctl -w net.ipv6.conf.all.disable_ipv6=1"
```

Step 11: Identifying Hidden Telemetry Connections

Use Datura to block specific domains and apps systematically, then monitor what breaks:

```bash
Approach: Whitelist model (block everything, then allow what's needed)

1. Block all apps except critical ones
adb shell cmd firewall set-uid-rule <PACKAGE> both BLOCK

2. Launch app and check if it connects
(Wait 30 seconds, most telemetry is eager)

3. Check system logs for connection attempts
adb shell logcat | grep -i "connect\|socket\|network"

4. Allow traffic to specific domains as needed
Use separate VPN with logging to identify required domains
```

This approach reveals which apps need which network access, enabling precise firewall rules.

Step 12: Datura Rule Persistence and Modifications

Firewall rules persist through reboots and app updates. However, system updates may reset Datura configuration:

```bash
Backup current Datura rules
adb shell dumpsys firewall > datura_rules_backup.txt

Document your configuration
This enables quick restoration if system update resets rules
```

Performance Impact Analysis

Datura operates at the kernel level, which is inherently more efficient than user-space VPN filters. However, inspect resource usage:

```bash
Check for memory overhead
adb shell dumpsys meminfo | grep -i "firewall\|datura"

Monitor CPU usage during active filtering
adb shell top -n 1 | grep -i "firewall"
```

Most systems show negligible overhead. If Datura consumes >5% CPU, there may be excessive policy enforcement overhead.

Troubleshooting Datura Connectivity Issues

When apps fail to connect after Datura configuration:

```bash
Step 1: Verify app UID
adb shell pm list packages -U | grep "com.example.app"

Step 2: Check current rules for that UID
adb shell su -c "iptables -L OUTPUT -n | grep <UID>"

Step 3: Monitor network events as app tries to connect
adb shell strace -e socket,connect -p <APP_PID>

Step 4: Review app manifest for required permissions
adb shell dumpsys packages | grep -A 20 "com.example.app"
```

These commands narrow down whether blocking is intentional (Datura rules) or accidental (permissions).

Step 13: Build Firewall Policies from Network Analysis

Use tcpdump to capture actual network traffic, then determine what to block:

```bash
Enable kernel packet capturing
adb shell su -c "tcpdump -i any -w /sdcard/capture.pcap &"

Let app run for 30 seconds
sleep 30

Pull capture and analyze
adb pull /sdcard/capture.pcap
wireshark capture.pcap  # Graphical analysis

From capture, identify:
- Remote IPs/domains contacted
- Ports used
- Protocols (HTTP, HTTPS, DNS, etc.)
- Frequency of connections
```

Use this data to create targeted blocking rules rather than blanket app blocks.

Frequently Asked Questions

How long does it take to controlling per-app?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [macOS Firewall Configuration](/macos-firewall-configuration-for-privacy/)
- [How VPN Interacts With Firewall Rules Iptables Nftables](/how-vpn-interacts-with-firewall-rules-iptables-nftables-guide/)
- [Configure Firewall Rules on OPNsense to Block Known](/how-to-configure-firewall-rules-on-opnsense-to-block-known-t/)
- [Calyxos Microg Setup Guide Getting Google Apps Working](/calyxos-microg-setup-guide-getting-google-apps-working-without-google-services/)
- [How To Bypass China Great Firewall Using Shadowsocks](/how-to-bypass-china-great-firewall-using-shadowsocks-with-ob/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
