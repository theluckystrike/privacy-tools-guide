---
layout: default
title: "Windows 11 Telemetry Disable Guide: Step by Step"
description: "A guide to disabling Windows 11 telemetry for developers and power users. Learn multiple methods including Settings, Group Policy"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /windows-11-telemetry-disable-guide-step-by-step/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
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

## Method 6: Disabling Windows Telemetry Services

Beyond Registry and Group Policy, disabling the actual services responsible for telemetry provides additional protection. Use Services Manager (`services.msc`) or PowerShell:

```powershell
# Disable Connected User Experiences and Telemetry Service
Set-Service -Name "DiagTrack" -StartupType Disabled
Stop-Service -Name "DiagTrack" -Force

# Disable dmwappushservice
Set-Service -Name "dmwappushservice" -StartupType Disabled
Stop-Service -Name "dmwappushservice" -Force

# Disable Update Orchestrator Service (manage carefully - impacts updates)
Set-Service -Name "UsoSvc" -StartupType Disabled
Stop-Service -Name "UsoSvc" -Force

# Disable App Diagnostics Service
Set-Service -Name "DiagSense" -StartupType Disabled
Stop-Service -Name "DiagSense" -Force

# List all running services to identify suspicious ones
Get-Service | Where-Object {$_.Status -eq "Running"} | Select-Object Name, DisplayName | Sort-Object Name
```

After disabling services, restart your computer. Some services will be disabled on reboot, while others may require additional Registry modifications to prevent re-enabling.

## Network-Level Telemetry Blocking

Some telemetry transmissions occur at the network level. You can block Windows telemetry hosts through your firewall or hosts file. This method supplements Registry changes and service disabling:

```powershell
# Add telemetry hosts to Windows hosts file
# First, open C:\Windows\System32\drivers\etc\hosts in Notepad with admin privileges

# Add these entries to block common telemetry endpoints:
# 0.0.0.0 telemetry.microsoft.com
# 0.0.0.0 settings-win.data.microsoft.com
# 0.0.0.0 vortex.data.microsoft.com
# 0.0.0.0 watson.telemetry.microsoft.com
# 0.0.0.0 settings.data.microsoft.com
```

For Windows Firewall users, create outbound rules blocking these domains:

```powershell
# Create firewall rule to block telemetry
New-NetFirewallRule -DisplayName "Block Windows Telemetry" `
    -Direction Outbound `
    -Action Block `
    -RemoteAddress @("13.107.4.50", "13.107.5.88", "40.77.226.189") `
    -Protocol TCP `
    -Enabled True
```

## Task Scheduler Telemetry Cleanup

Windows schedules numerous telemetry tasks through Task Scheduler. Disabling these prevents them from running automatically:

```powershell
# Disable common telemetry tasks
$telemetryTasks = @(
    "Microsoft\Windows\Application Experience\ProgramDataUpdater",
    "Microsoft\Windows\Application Experience\StartupTask",
    "Microsoft\Windows\Autochk\Proxy",
    "Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
    "Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",
    "Microsoft\Windows\DiskDiagnostic\Microsoft-Windows-DiskDiagnosticDataCollector",
    "Microsoft\Windows\Shell\CreateObjectTask",
    "Microsoft\Windows\System Restore\SR",
    "Microsoft\Windows\Diagnosis\RecommendedTroubleshootingScanner"
)

foreach ($task in $telemetryTasks) {
    Disable-ScheduledTask -TaskName $task -ErrorAction SilentlyContinue
}
```

## Verification and Monitoring

After applying your chosen method, verify the configuration:

```powershell
# Check current telemetry level
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "TelemetryAllow" -ErrorAction SilentlyContinue | Select-Object -ExpandProperty TelemetryAllow

# Check status of telemetry services
Get-Service DiagTrack, dmwappushservice, UsoSvc, DiagSense | Format-Table Name, Status, StartType

# Monitor outbound connections to known telemetry endpoints
Get-NetTCPConnection -State Established | Where-Object { $_.RemoteAddress -match "^(13\.|40\.|52\.)" } | Format-Table RemoteAddress, OwningProcess, OwningModule

# Check disabled scheduled tasks
Get-ScheduledTask | Where-Object {$_.State -eq "Disabled"} | Select-Object TaskPath, TaskName
```

Additional monitoring involves checking Event Viewer under Applications and Services Logs → Microsoft → Windows → Diagnostics → Diagnostic-Performance for telemetry-related events. If you've disabled services properly, these logs should be sparse.

For advanced network monitoring, use **Wireshark** or **NetFlow** to capture outbound traffic and verify that no connections are being made to Microsoft telemetry endpoints.

## Advanced: Group Policy for Domain-Joined Machines

For users in organizations or those managing multiple machines, Group Policy provides centralized telemetry control. Edit the Group Policy Object (GPO) to enforce consistent telemetry settings across all managed computers:

```powershell
# Domain administrators: Import GPO template for telemetry control
# 1. Download Windows 11 ADMX templates from Microsoft
# 2. Copy to %WINDIR%\PolicyDefinitions\

# Create new GPO for telemetry control
New-GPO -Name "TelemetryDisable" -Comment "Disable Windows 11 Telemetry"

# Link GPO to organizational unit
New-GPLink -Name "TelemetryDisable" -Target "OU=Computers,DC=example,DC=com"

# Force policy update on managed computers
gpupdate /force

# Verify policy application
gpresult /h report.html
```

## Understanding What Gets Disabled vs. What Stays

Important to note what remains operational after disabling telemetry:

**Still works normally:**
- Windows Update and patches
- Microsoft Defender antivirus
- Windows Security essentials
- Driver updates
- Store app functionality (mostly)
- Cortana (but with reduced features)
- Task scheduling and automation

**May be affected:**
- Diagnostic reports (useful for troubleshooting)
- Customer Experience Improvement Program (CEIP)
- Windows Spotlight features
- App recommendations
- Tailored experience suggestions

**Completely disabled:**
- Activity history collection
- Web and app activity tracking
- Targeted ads and personalization
- Search indexing for personalization
- Suggested content

## Persistent Telemetry Issues and Workarounds

Despite your best efforts, Windows may re-enable telemetry after major updates. Use these strategies for persistent configuration:

```powershell
# Create a scheduled task to enforce telemetry settings on reboot
$action = New-ScheduledTaskAction -Execute powershell.exe `
    -Argument "-ExecutionPolicy Bypass -File C:\scripts\disable-telemetry.ps1"
$trigger = New-ScheduledTaskTrigger -AtStartup
$principal = New-ScheduledTaskPrincipal -UserID "NT AUTHORITY\SYSTEM" -RunLevel Highest
Register-ScheduledTask -TaskName "DisableTelemetryAtBoot" -Action $action -Trigger $trigger -Principal $principal

# Alternatively, disable Windows Update temporarily between policy applications
# WARNING: Only do this if you can apply patches manually

Stop-Service -Name "wuauserv" -Force
Set-Service -Name "wuauserv" -StartupType Disabled
```

Create a PowerShell script stored in a protected location and set it to run at startup with elevated privileges. This ensures telemetry settings persist even after Windows Updates attempt to reset them.

## Important Considerations

Disabling telemetry may affect certain Windows features. Some applications rely on diagnostic data for functionality, and certain update mechanisms verify system health through telemetry channels. After disabling, test critical functionality including Windows Update, Microsoft Store apps, and any connected services.

Windows Defender and core security features continue functioning regardless of telemetry settings. Security-related data collection operates independently of the diagnostic telemetry system.

For developers building applications that integrate with Windows telemetry APIs, be aware that reduced telemetry may limit available diagnostic data for your applications. Test thoroughly in environments matching your configured telemetry level.

One important note: Disabling Update Orchestrator Service (`UsoSvc`) can interfere with Windows Updates. If you disable this service, use Windows Update manually through Settings or configure WSUS if managing multiple machines in an organization.




## Related Articles

- [How To Disable All Windows 11 Telemetry Endpoints Using Fire](/privacy-tools-guide/how-to-disable-all-windows-11-telemetry-endpoints-using-fire/)
- [Windows 11 Privacy Settings: How to Disable Telemetry](/privacy-tools-guide/windows-11-privacy-settings-disable-telemetry/)
- [How To Disable Smart App Control In Windows 11 That Reports](/privacy-tools-guide/how-to-disable-smart-app-control-in-windows-11-that-reports-/)
- [Windows 11 Cortana Disable Privacy Guide](/privacy-tools-guide/windows-11-cortana-disable-privacy-guide/)
- [Windows Activity History Disable Privacy Guide](/privacy-tools-guide/windows-activity-history-disable-privacy-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
