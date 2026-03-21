---
layout: default
title: "Windows 11 Privacy Settings: How to Disable Telemetry"
description: "A practical guide for developers and power users to disable Windows 11 telemetry through Settings, Group Policy, and Registry modifications"
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /windows-11-privacy-settings-disable-telemetry/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Reduce Windows 11 telemetry by changing diagnostic data from Enhanced/Full to Basic through Settings (Privacy & security → Diagnostics & feedback), disabling Tailored experiences and Feedback frequency. For Pro/Enterprise editions, use Group Policy (gpldit.msc) to set Computer Configuration → Administrative Templates → Windows Components → Data Collection and On-Demand Diagnostics to Diagnostic data off (minimum level). Home edition users must edit Registry (HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\DataCollection) and disable telemetry services manually. Complete telemetry blocking requires disabling multiple background services (DiagTrack, dmwappushservice) and blocking Microsoft's telemetry endpoints at the firewall level, which is more aggressive but removes crash reporting and usage statistics sent to Microsoft.

## Understanding Windows 11 Telemetry Levels

Windows 11 offers four telemetry levels: Security, Basic, Enhanced, and Full. The default setting on most systems is Enhanced or Full, which sends substantial diagnostic data including crash reports, usage statistics, and app performance metrics.

To check your current telemetry level, open Settings → Privacy & security → Windows security → Diagnostics & feedback. However, accessing this through the GUI limits your control. Power users prefer programmatic methods for repeatable, auditable configurations.

## Method 1: Using Settings App (Basic)

The simplest approach uses the Windows Settings application:

1. Press `Win + I` to open Settings
2. Navigate to Privacy & security → Windows security → Diagnostics & feedback
3. Set "Diagnostic data" to Minimum or Basic
4. Disable "Tailored experiences" and "Feedback frequency"

While this method works for average users, it does not fully disable telemetry. Microsoft still collects essential diagnostic data required for security updates.

## Method 2: Using Group Policy Editor (Recommended for Pro/Enterprise)

Windows 11 Pro and Enterprise users can use Group Policy for more control. This method requires administrative privileges.

Open Group Policy Editor by pressing `Win + R`, typing `gpedit.msc`, and pressing Enter. Navigate to:

```
Computer Configuration → Administrative Templates → Windows Components → Data Collection and Preview Builds
```

Locate "Allow Telemetry" and set it to Disabled. This disables all telemetry except the Security level required for Windows Update.

For more granular control, you can set it to "1 - Basic" for minimal data collection while maintaining compatibility with enterprise management tools.

## Method 3: Registry Modifications (All Editions)

For Windows 11 Home users who cannot access Group Policy, direct Registry modifications provide equivalent functionality. This method requires creating a backup before making changes.

Create a file named `disable-telemetry.reg` with the following content:

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\DataCollection]
"AllowTelemetry"=dword:00000000
```

Double-click the file to apply it, or import it via command line:

```cmd
reg import disable-telemetry.reg
```

This sets telemetry to the minimum level (0), equivalent to the Security level.

## Method 4: PowerShell Automation Script

For developers who manage multiple machines or want reproducible configurations, PowerShell provides the most flexible approach. Here's a script:

```powershell
# Disable-WindowsTelemetry.ps1
# Run as Administrator

# Disable telemetry via Registry
$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection"
if (!(Test-Path $registryPath)) {
    New-Item -Path $registryPath -Force | Out-Null
}
Set-ItemProperty -Path $registryPath -Name "AllowTelemetry" -Value 0 -Type DWord

# Disable connected user experience
$cuPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent"
if (!(Test-Path $cuPath)) {
    New-Item -Path $cuPath -Force | Out-Null
}
Set-ItemProperty -Path $cuPath -Name "DisableWindowsConsumerFeatures" -Value 1 -Type DWord

# Disable feedback notifications
$feedbackPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection\Preview"
if (!(Test-Path $feedbackPath)) {
    New-Item -Path $feedbackPath -Force | Out-Null
}
Set-ItemProperty -Path $feedbackPath -Name "AllowPreviewFlighting" -Value 0 -Type DWord

Write-Host "Telemetry disabled. Restart required for changes to take effect."
```

Save this script and execute it with elevated privileges:

```powershell
.\Disable-WindowsTelemetry.ps1
```

## Method 5: Using Windows Spy Block or Similar Tools

Several open-source tools automate these configurations. Windows Spy Block (available on GitHub) applies privacy-preserving settings across multiple categories including telemetry, search, and cloud integration.

For developers comfortable with scripts, manually applying each setting provides better understanding and control. However, automated tools save time when configuring fresh Windows installations.

## Additional Privacy Settings to Consider

Beyond telemetry, several related settings affect data collection:

**Activity History**: Navigate to Settings → Privacy & security → Activity history and disable "Send my activity history to Microsoft." This prevents syncing of app usage and timeline data.

**Location**: Disable location services at Settings → Privacy & security → Location if not required. Many apps request location access unnecessarily.

**Tailored Experiences**: Disable personalized recommendations at Settings → Privacy & security → Diagnostics & feedback. This setting uses your diagnostic data to provide targeted suggestions.

**Search Permissions**: In Settings → Privacy & security → Search permissions, disable "Cloud search" and limit search history retention.

## Verifying Your Configuration

After applying changes, verify telemetry settings using this PowerShell command:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name AllowTelemetry
```

A value of `0` indicates minimum telemetry. You can also monitor network connections using tools like Process Monitor or Wireshark to confirm reduced outbound connections to Microsoft endpoints.

## Telemetry Service Hardening

Beyond Registry and Group Policy changes, several background services collect telemetry. Disabling these services provides additional privacy:

```powershell
# DiagTrack: Connected User Experience and Telemetry Service
# This is one of the most aggressive telemetry services
Stop-Service DiagTrack -NoWait
Set-Service DiagTrack -StartupType Disabled

# dmwappushservice: Device Management Wireless Application Protocol (WAP)
# Pushes configuration and telemetry to devices
Stop-Service dmwappushservice -NoWait
Set-Service dmwappushservice -StartupType Disabled

# OneSyncSvc: Microsoft account synchronization
# Syncs settings, app data, and browsing history to Microsoft account
Stop-Service OneSyncSvc -NoWait
Set-Service OneSyncSvc -StartupType Disabled

# Verify services are disabled
Get-Service DiagTrack, dmwappushservice, OneSyncSvc | Select Status
# Output should show all as Stopped
```

Caution: Disabling services can affect functionality. DiagTrack in particular may be required for Windows Update in some scenarios. If you encounter issues, re-enable with `Set-Service ServiceName -StartupType Manual`.

## Firewall Rules for Telemetry Endpoints

For maximum privacy, block Microsoft's telemetry endpoints at the firewall level:

```powershell
# Common Microsoft telemetry endpoints
$endpoints = @(
  "telemetry.microsoft.com",
  "telemetry-inm.trafficmanager.net",
  "vortex.data.microsoft.com",
  "telemetry.appex.bing.net",
  "watson.telemetry.microsoft.com",
  "ceuswatson.telemetry.microsoft.com",
  "telemetry.urs.microsoft.com",
  "wd-prod-ms-winttd.trafficmanager.net",
  "settings-win.data.microsoft.com"
)

foreach ($endpoint in $endpoints) {
  # Create outbound firewall rule to block DNS resolution
  New-NetFirewallRule -DisplayName "Block $endpoint" `
    -Direction Outbound `
    -Action Block `
    -RemoteAddress $endpoint `
    -Protocol TCP, UDP
}
```

Run this as Administrator. Verify rules were created:

```powershell
Get-NetFirewallRule -DisplayName "Block *" | Format-Table DisplayName, Enabled
```

This approach is aggressive—some Windows features may break. For fine-grained control, use firewall rules selectively on the most intrusive endpoints (telemetry.microsoft.com is the primary concern).

## Monitoring Network Activity

To verify telemetry is actually disabled, monitor network connections:

```powershell
# Install netsh if needed (typically pre-installed)
# Use netstat to identify established connections to Microsoft domains

netstat -ano | findstr "telemetry\|watson\|vortex"
```

For graphical monitoring, use third-party tools:
- **Wireshark**: Packet-level analysis showing exactly what data leaves your computer
- **Process Monitor** (Sysinternals): Monitor which processes make network connections
- **NetLimiter**: GUI-based traffic monitoring with per-process control

Run these tools before and after applying telemetry disabling to verify reduction in outbound connections.

## Application-Level Privacy Settings

Beyond OS-level telemetry, individual Windows apps collect data:

**Windows Update**: Settings → Update & security → Advanced options
- Disable "Get the latest updates as soon as they're available"
- Disable "Optional diagnostics data"

**Microsoft Edge**: Settings → Privacy → Clear browsing data
- Disable "Save browsing history"
- Disable "Allow sites to check if you have payment methods saved"
- Disable "Improve your browsing experience"

**Windows Defender**: Windows security → Virus & threat protection
- Disable "Cloud-delivered protection" if you use alternative antivirus
- However, many users rely on Defender—disabling requires alternative solution

**OneDrive**: Settings → Cloud sync options
- Uncheck "Automatically save screenshots"
- Uncheck "Automatically save things I copy"

These app-level settings provide granular control over specific data collection.

## Verifying Privacy Settings with Third-Party Tools

Several tools help audit Windows privacy settings:

**W10Privacy**: Open-source tool that visualizes privacy settings and allows batch changes. Available on GitHub.

**PrivateZilla**: Similar to W10Privacy but with cleaner UI. Also open-source.

**O&O ShutUp++**: Commercial tool (paid) with privacy auditing and recommendations.

**PolicyPlus**: Group Policy administration tool for Home edition users. Less intuitive but .

Use these tools to audit settings after manual configuration. They help identify missed settings or services that still have telemetry enabled.

## Impact on Windows Features

Disabling telemetry has side effects. Understanding them helps make informed decisions:

**Windows Update**: Relies on diagnostic data to troubleshoot update failures. Severe telemetry disabling may prevent rollback on failed updates.

**Windows Defender**: Cloud-based threat intelligence requires diagnostic data. Disabling may reduce threat detection effectiveness. Consider using alternative antivirus (Bitdefender, Kaspersky) with better privacy practices.

**Game Bar and Xbox integration**: Heavily integrated with telemetry. Disabling telemetry may break these features.

**Cortana**: The voice assistant relies extensively on telemetry. Disable if privacy is priority.

**Windows Store Apps**: First-party apps (Mail, Photos, Weather) require telemetry for core functionality. Consider using third-party alternatives (Thunderbird for email, VLC for media).

## Trade-offs to Consider

Disabling telemetry has implications. Some features may degrade: Windows Update troubleshooting relies on diagnostic data, and certain enterprise management capabilities require enhanced telemetry. Additionally, some first-party apps (like Photos or Weather) may show reduced functionality.

For most developers and power users, the privacy benefits outweigh these trade-offs. Security-conscious organizations often disable telemetry as part of baseline hardening procedures.

Consider your specific use case:
- **Personal use**: Aggressive telemetry disabling is appropriate
- **Corporate environment**: Coordinate with IT—some telemetry may be required for device management
- **Server/headless system**: Disable everything—servers don't need user-facing telemetry
- **Gaming**: Be conservative—Game Bar and Xbox integration provide useful features

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
