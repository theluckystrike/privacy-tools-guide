---

layout: default
title: "Windows 11 Privacy Settings: How to Disable Telemetry"
description: "A practical guide for developers and power users to disable Windows 11 telemetry through Settings, Group Policy, and Registry modifications."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /windows-11-privacy-settings-disable-telemetry/
categories: [guides]
reviewed: true
score: 8
---

{% raw %}

Windows 11 collects significant amounts of telemetry data by default, sending diagnostic information to Microsoft servers. For developers and power users who value privacy or work in security-sensitive environments, understanding how to minimize this data collection is essential. This guide covers multiple methods to disable or reduce Windows 11 telemetry, from basic Settings adjustments to advanced Group Policy and Registry configurations.

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

Windows 11 Pro and Enterprise users can use Group Policy for more comprehensive control. This method requires administrative privileges.

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

For developers who manage multiple machines or want reproducible configurations, PowerShell provides the most flexible approach. Here's a comprehensive script:

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

## Trade-offs to Consider

Disabling telemetry has implications. Some features may degrade: Windows Update troubleshooting relies on diagnostic data, and certain enterprise management capabilities require enhanced telemetry. Additionally, some first-party apps (like Photos or Weather) may show reduced functionality.

For most developers and power users, the privacy benefits outweigh these trade-offs. Security-conscious organizations often disable telemetry as part of baseline hardening procedures.

## Conclusion

Windows 11 telemetry collection is extensive but controllable. Whether you use the basic Settings approach, Group Policy for enterprise environments, or PowerShell scripts for automation, you can significantly reduce data sent to Microsoft. Start with the Settings method for simplicity, then advance to Registry or Group Policy configurations for more complete control. Regular review of Windows privacy settings remains advisable as Microsoft updates telemetry behavior with each feature update.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
