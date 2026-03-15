---

layout: default
title: "Windows 11 Telemetry Disable Guide: Step by Step"
description: "A comprehensive guide to disabling Windows 11 telemetry for developers and power users. Learn multiple methods including Settings, Group Policy, Registry, and PowerShell."
date: 2026-03-15
author: theluckystrike
permalink: /windows-11-telemetry-disable-guide-step-by-step/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Windows 11 collects significant telemetry data by default, sending information about your system usage, hardware configuration, and application behavior to Microsoft. For developers and power users who prioritize privacy or need to minimize network traffic, disabling telemetry requires navigating multiple configuration layers. This guide provides practical methods to reduce or eliminate Windows 11 telemetry collection.

## Understanding Windows 11 Telemetry Levels

Windows 11 offers four telemetry levels: Security, Basic, Enhanced, and Full. The Security level sends only data required for Windows security, while Full transmits extensive diagnostic information. Most users default to Enhanced or Full, making privacy-conscious configuration essential.

To check your current telemetry level, open Settings → Privacy & security → Windows security → Open Windows Security, then navigate to Settings → Privacy and diagnostics. Alternatively, use this PowerShell command:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "TelemetryAllow" -ErrorAction SilentlyContinue
```

## Method 1: Using Windows Settings

The simplest approach uses the Settings application. Open Settings → Privacy & security → Windows security → Open Windows Security, then click Settings for Microsoft Defender and scroll to Diagnostic data. Set this to Required diagnostic data only.

For additional reduction, navigate to Settings → Privacy & security → Activity history and disable all options including Activity history, Show activities from accounts, and Delete activity history.

These settings provide basic reduction but do not eliminate all telemetry. Microsoft still collects essential security data and certain diagnostic information required for Windows updates.

## Method 2: Using Local Group Policy Editor

The Group Policy Editor provides more granular control. Press Win + R, type `gpedit.msc`, and navigate to Computer Configuration → Administrative Templates → Windows Components → Data Collection and Preview Builds.

Double-click on Telemetry and set it to Enabled, then select 0 - Security from the dropdown. This setting forces the minimum telemetry level across all Windows components.

For organizations or users requiring complete control, additional policies exist under Computer Configuration → Administrative Templates → System → Internet Communication Management. Configure these to restrict various Windows communication features:

```powershell
# Disable CEIP (Customer Experience Improvement Program)
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\SQMClient\Windows" -Name "CEIPEnable" -Value 0 -PropertyType DWord -Force
```

## Method 3: Using Registry Modifications

For systems without Group Policy Editor (such as Windows 11 Home), direct Registry modifications achieve similar results. Create a backup before making changes, then run Registry Editor (regedit) and navigate to the following keys:

Set telemetry to minimum by creating or modifying this key:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "TelemetryAllow" -Value 0 -PropertyType DWord -Force
```

Additional Registry settings further reduce telemetry:

```powershell
# Disable Feedback Hub telemetry
New-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Privacy" -Name "TailoredExperiencesWithDiagnosticDataEnabled" -Value 0 -PropertyType DWord -Force

# Disable diagnostic data at system level
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Name "MaxTelemetryAllowed" -Value 1 -Type DWord

# Disable Windows Tips
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" -Name "DisableWindowsSpotlightFeatures" -Value 1 -PropertyType DWord -Force
```

## Method 4: Using PowerShell Automation

PowerShell provides the most reproducible method for disabling telemetry. Create a script that applies multiple settings simultaneously:

```powershell
# Windows 11 Telemetry Disable Script
# Run as Administrator

# Set telemetry to minimum via Registry
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "TelemetryAllow" -Value 0 -Type DWord -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Name "MaxTelemetryAllowed" -Value 1 -Type DWord

# Disable Activity History
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Privacy" -Name "ActivityHistoryEnabled" -Value 0 -Type DWord -Force

# Disable feedback notifications
New-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\ContentDeliveryManager" -Name "SubscribedContent-338387Enabled" -Value 0 -PropertyType DWord -Force

# Disable Tailored Experiences
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Privacy" -Name "TailoredExperiencesWithDiagnosticDataEnabled" -Value 0 -Type DWord -Force

# Disable CEIP
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\SQMClient\Windows" -Name "CEIPEnable" -Value 0 -PropertyType DWord -Force

Write-Host "Telemetry settings applied. Restart required for full effect."
```

Save this as `disable-telemetry.ps1` and execute with administrator privileges. After running, restart your computer for all changes to take effect.

## Method 5: Using Windows Terminal Services (Enterprise Environments)

For enterprise deployments, Windows Terminal Services and mobile device management solutions offer centralized telemetry control. Configure these through Microsoft Intune or third-party management tools by creating configuration profiles that enforce telemetry levels across all managed devices.

Corporate users should verify their organization's policies before making changes, as some telemetry data supports security updates and compliance requirements.

## Verification and Monitoring

After applying your chosen method, verify the configuration:

```powershell
# Check current telemetry level
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "TelemetryAllow" -ErrorAction SilentlyContinue | Select-Object -ExpandProperty TelemetryAllow

# Monitor outbound connections (requires Network Monitor or similar)
# This shows processes making external connections
Get-NetTCPConnection -State Established | Where-Object { $_.RemoteAddress -like "13.*" -or $_.RemoteAddress -like "52.*" } | Format-Table RemoteAddress, OwningProcess
```

Additional monitoring involves checking Event Viewer under Applications and Services Logs → Microsoft → Windows → Diagnostics → Diagnostic-Performance for telemetry-related events.

## Important Considerations

Disabling telemetry may affect certain Windows features. Some applications rely on diagnostic data for functionality, and certain update mechanisms verify system health through telemetry channels. After disabling, test critical functionality including Windows Update, Microsoft Store apps, and any connected services.

Windows Defender and core security features continue functioning regardless of telemetry settings. Security-related data collection operates independently of the diagnostic telemetry system.

For developers building applications that integrate with Windows telemetry APIs, be aware that reduced telemetry may limit available diagnostic data for your applications. Test thoroughly in environments matching your configured telemetry level.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
