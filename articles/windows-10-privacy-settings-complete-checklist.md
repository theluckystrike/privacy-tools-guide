---
layout: default
title: "Windows 10 Privacy Settings Complete Checklist"
description: "Windows 10 collects significant telemetry data by default, and while Microsoft has softened some practices since its initial release, privacy-conscious users"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /windows-10-privacy-settings-complete-checklist/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]---
---
layout: default
title: "Windows 10 Privacy Settings Complete Checklist"
description: "Windows 10 collects significant telemetry data by default, and while Microsoft has softened some practices since its initial release, privacy-conscious users"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /windows-10-privacy-settings-complete-checklist/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]---

{% raw %}

Windows 10 collects significant telemetry data by default, and while Microsoft has softened some practices since its initial release, privacy-conscious users and developers still need to manually configure numerous settings. This checklist covers essential privacy configurations, registry tweaks, and PowerShell commands that reduce data collection without breaking system functionality.

## Key Takeaways

- **Do these recommendations work**: for small teams? Yes, most practices scale down well.
- **Open Settings > Privacy**: & Security and work through these critical areas: General Privacy Turn off "Let apps use advertising ID" to prevent apps from tracking you across apps and websites.
- **Navigate to OneDrive >**: Settings > Backup > Manage backup and deselect all folders if you prefer local-only storage.
- **Disable "Let websites provide**: locally relevant content" if you prefer not to share your language preferences with web services.
- **This prevents Windows from**: storing which applications you use and when.
- **Most productivity applications have**: no legitimate reason to access your camera or microphone.

## Disable Telemetry Through Settings

The first layer of privacy protection exists in Windows Settings. Open **Settings > Privacy & Security** and work through these critical areas:

**General Privacy**
Turn off "Let apps use advertising ID" to prevent apps from tracking you across apps and websites. Disable "Let websites provide locally relevant content" if you prefer not to share your language preferences with web services.

**Diagnostics & Feedback**
Set "Diagnostic data" to **Minimum** rather than Required or Optional. Disable "Tailored experiences" and "Diagnostic data viewer" entirely. Under "Feedback frequency," select "Never" to stop the recurring prompts.

**Activity History**
Uncheck "Send my activity history to Microsoft" and clear any existing history. This prevents Windows from storing which applications you use and when.

**Location Services**
Disable location for any apps that don't require it. For developers building location-aware applications, create a separate user account for testing to isolate location data from your primary workflow.

**Camera and Microphone Access**
Navigate to Settings > Privacy > Camera and review every app listed. Most productivity applications have no legitimate reason to access your camera or microphone. Disable access for browsers, text editors, and any application that doesn't explicitly require it. The built-in Camera app and communication tools are the only justified exceptions on a developer workstation.

**App Permissions Audit**
Work through all permission categories: Contacts, Calendar, Call History, Email, Tasks, Messaging, Radios, and Other Devices. Windows 10 grants many of these by default, but nearly no desktop productivity software needs access to your contacts or calendar through the OS permission system. Revoke access liberally and re-grant only when an application specifically requests and justifies it.

## Disable Telemetry via Group Policy and Registry

For enterprise and professional editions, Group Policy provides additional controls. Run `gpedit.msc` and navigate to:

```
Computer Configuration > Administrative Templates > Windows Components > Data Collection and Preview Builds
```

Set "Allow Telemetry" to **0** (Security) or **1** (Basic) depending on your security requirements. The Security level sends only security-related data required for Windows Update.

For Home editions without Group Policy, apply registry modifications. Create a backup before editing the registry:

```powershell
# Export current registry state
reg export "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" backup_telemetry.reg

# Disable telemetry
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
```

Additional registry keys worth addressing include the advertising ID and app launch tracking:

```powershell
# Disable advertising ID
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo" /v Enabled /t REG_DWORD /d 0 /f

# Disable app launch tracking
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v Start_TrackProgs /t REG_DWORD /d 0 /f

# Disable cloud clipboard (syncs clipboard history to Microsoft)
reg add "HKCU\Software\Microsoft\Clipboard" /v EnableClipboardHistory /t REG_DWORD /d 0 /f
```

## PowerShell Scripts for Complete Privacy

PowerShell automation improves privacy configuration across multiple machines. The following script disables common telemetry services:

```powershell
# Run as Administrator
$services = @(
 "DiagTrack", # Connected User Experiences and Telemetry
 "dmwappushservice", # WAP Push Service
 "WbioSrvc", # Windows Biometric Service (if unused)
 "OneSyncSvc", # Sync Service
 "RemoteRegistry" # Remote Registry
)

foreach ($service in $services) {
 $svc = Get-Service -Name $service -ErrorAction SilentlyContinue
 if ($svc) {
 Stop-Service -Name $service -Force -ErrorAction SilentlyContinue
 Set-Service -Name $service -StartupType Disabled -ErrorAction SilentlyContinue
 Write-Host "Disabled: $service"
 }
}
```

To prevent Windows Update from reenabling telemetry services, create a scheduled task:

```powershell
# Create task to monitor and disable telemetry weekly
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-Command `"Get-Service DiagTrack | Set-Service -StartupType Disabled -ErrorAction SilentlyContinue`""
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 3am
Register-ScheduledTask -Action $action -Trigger $trigger -Name "Privacy-Monitor" -Description "Disables telemetry services weekly" -Force
```

Additional telemetry-related scheduled tasks that Windows creates by default should also be disabled. Microsoft plants several tasks in the Task Scheduler that run independently of service states:

```powershell
# Disable telemetry-related scheduled tasks
$tasksToDisable = @(
    "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser",
    "\Microsoft\Windows\Application Experience\ProgramDataUpdater",
    "\Microsoft\Windows\Autochk\Proxy",
    "\Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
    "\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",
    "\Microsoft\Windows\DiskDiagnostic\Microsoft-Windows-DiskDiagnosticDataCollector",
    "\Microsoft\Windows\Feedback\Siuf\DmClient"
)

foreach ($task in $tasksToDisable) {
    Disable-ScheduledTask -TaskPath (Split-Path $task) -TaskName (Split-Path $task -Leaf) -ErrorAction SilentlyContinue
    Write-Host "Disabled task: $task"
}
```

## Network-Level Blocking

For developers running local networks or managing DNS, block telemetry at the network level. Add these entries to your hosts file (located at `C:\Windows\System32\drivers\etc\hosts`):

```
0.0.0.0 v10.vortex-win.data.microsoft.com
0.0.0.0 settings-win.data.microsoft.com
0.0.0.0 watson.telemetry.microsoft.com
0.0.0.0 sqm.telemetry.microsoft.com
0.0.0.0 telemetry.microsoft.com
0.0.0.0 win10.ipv6.microsoft.com
0.0.0.0 config.ads.moatads.com
```

On Windows 10 Pro or Enterprise, configure Windows Firewall to block all outbound connections to Microsoft telemetry endpoints:

```powershell
# Block telemetry via Windows Firewall
$rules = @(
 @{Name="Block-Telemetry-1"; Prog="System"; Dir="Outbound"; RemotePort="443"; RemoteAddr="v10.vortex-win.data.microsoft.com"},
 @{Name="Block-Telemetry-2"; Prog="System"; Dir="Outbound"; RemotePort="443"; RemoteAddr="settings-win.data.microsoft.com"}
)

foreach ($rule in $rules) {
 New-NetFirewallRule -DisplayName $rule.Name -Direction $rule.Dir `
 -Program $rule.Prog -RemotePort $rule.RemotePort `
 -RemoteAddress $rule.RemoteAddr -Action Block -Enabled True -ErrorAction SilentlyContinue
}
```

For home users managing Pi-hole or similar network-level DNS filtering, a curated blocklist specifically targeting Microsoft telemetry domains works more reliably than hosts file entries because it catches IPv6 fallback addresses and CDN-routed telemetry that hosts files miss. The hagezi and StevenBlack blocklists both include Microsoft-specific categories.

## Disable Cortana and Search Indexing

Cortana collects voice data and search queries. Disable it through Group Policy or registry:

```powershell
# Disable Cortana via registry
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowCortana /t REG_DWORD /d 0 /f

# Disable Windows Search indexing (frees resources, reduces data)
Set-Service -Name "WSearch" -StartupType Disabled
Stop-Service -Name "WSearch" -Force
```

After disabling Windows Search, expect slower file searches. Consider using Everything or alternative search tools for rapid file discovery.

To prevent Bing search results appearing in the Start menu search box — a privacy concern because queries are sent to Microsoft servers:

```powershell
# Disable web search in Start menu
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Search" /v BingSearchEnabled /t REG_DWORD /d 0 /f
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Search" /v CortanaConsent /t REG_DWORD /d 0 /f
```

## OneDrive and Cloud Sync Privacy

Windows 10 aggressively pushes OneDrive integration. Even if you don't actively use it, OneDrive can sync Desktop, Documents, and Pictures folders to Microsoft servers without explicit warning during the initial setup wizard.

```powershell
# Prevent OneDrive from starting automatically
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "OneDrive" /t REG_SZ /d "" /f

# Disable OneDrive via Group Policy (Enterprise/Pro)
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\OneDrive" /v DisableFileSyncNGSC /t REG_DWORD /d 1 /f
```

Check which folders are configured for cloud backup in OneDrive settings. Navigate to OneDrive > Settings > Backup > Manage backup and deselect all folders if you prefer local-only storage. Even with cloud backup disabled, OneDrive may still upload "Known Folder Move" data unless explicitly prevented at the registry level.

## Privacy Settings for Developers

Developers require additional considerations beyond standard privacy hardening.

**Visual Studio and SDK Telemetry**
Visual Studio collects usage data through the VS Feedback system. Disable it via:
- VS Installer > More > Disable sending feedback
- Registry: `HKCU\Software\Microsoft\VisualStudio\Feedback` set `IsEnabled` to `0`

**.NET CLI Telemetry**
The .NET CLI sends telemetry to Microsoft by default. This includes which commands you run, error information, and project types. Disable it with an environment variable:

```powershell
# Set permanently in system environment variables
[System.Environment]::SetEnvironmentVariable("DOTNET_CLI_TELEMETRY_OPTOUT", "1", "Machine")

# Or set for current session only
$env:DOTNET_CLI_TELEMETRY_OPTOUT = "1"
```

Many other developer tools have similar opt-outs: npm has `npm config set send-telemetry false`, Angular CLI has `ng analytics off`, and Homebrew on WSL respects `HOMEBREW_NO_ANALYTICS=1`.

**Windows Terminal Settings**
Configure Windows Terminal to avoid sending diagnostics:

```json
{
 "settings": {
 "diagnostics": {
 "pointerTelemetry": false
 },
 "privacy": {
 "sendCursorFeatures": false,
 "sendModulePaths": false
 }
 }
}
```

**Git Credential Manager**
Prevent Git from sending analytics:

```bash
git config --global credential.helper ""
git config --global core.askPass ""
git config --global http.postBuffer 524288000
```

## Verification and Ongoing Maintenance

After applying these settings, verify your configuration:

```powershell
# Check telemetry service status
Get-Service DiagTrack | Select-Object Name, Status, StartType

# Check firewall rules
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*Telemetry*"} | Select-Object Name, Enabled, Action

# Test network blocks
Test-NetConnection -ComputerName "v10.vortex-win.data.microsoft.com" -Port 443 -WarningAction SilentlyContinue
```

Microsoft periodically updates Windows with new telemetry endpoints. Create a monthly review habit:

```powershell
# Quick privacy health check script
$issues = @()

if ((Get-Service DiagTrack).StartType -ne "Disabled") {
 $issues += "DiagTrack is not disabled"
}

$hostsContent = Get-Content "C:\Windows\System32\drivers\etc\hosts"
if ($hostsContent -notmatch "telemetry.microsoft.com") {
 $issues += "Hosts file missing telemetry blocks"
}

if ($issues) {
 Write-Host "Privacy issues found:" $issues
} else {
 Write-Host "Privacy configuration looks good"
}
```

Major Windows feature updates (the twice-yearly releases) routinely reset some privacy settings to defaults. After every major update, re-run your privacy configuration scripts and verify all settings remain in place. Keep your PowerShell scripts version-controlled so re-applying them takes minutes rather than hours. This is especially important for enterprise environments where IT may also push Group Policy updates that conflict with privacy configurations.

## Frequently Asked Questions

**How do I prioritize which recommendations to implement first?**

Start with changes that require the least effort but deliver the most impact. Quick wins build momentum and demonstrate value to stakeholders. Save larger structural changes for after you have established a baseline and can measure improvement.

**Do these recommendations work for small teams?**

Yes, most practices scale down well. Small teams can often implement changes faster because there are fewer people to coordinate. Adapt the specifics to your team size—a 5-person team does not need the same formal processes as a 50-person organization.

**How do I measure whether these changes are working?**

Define 2-3 measurable outcomes before you start. Track them weekly for at least a month to see trends. Common metrics include response time, completion rate, team satisfaction scores, and error frequency. Avoid measuring too many things at once.

**Can I customize these recommendations for my specific situation?**

Absolutely. Treat these as starting templates rather than rigid rules. Every team and project has unique constraints. Test each recommendation on a small scale, observe results, and adjust the approach based on what actually works in your context.

**What is the biggest mistake people make when applying these practices?**

Trying to change everything at once. Pick one or two practices, implement them well, and let the team adjust before adding more. Gradual adoption sticks better than wholesale transformation, which often overwhelms people and gets abandoned.

## Related Articles

- [Windows 11 Privacy Settings: How to Disable Telemetry](/privacy-tools-guide/windows-11-privacy-settings-disable-telemetry/)
- [Windows Group Policy Privacy Settings Guide](/privacy-tools-guide/windows-group-policy-privacy-settings-guide/)
- [Windows OneDrive Privacy Settings Guide 2026](/privacy-tools-guide/windows-onedrive-privacy-settings-guide-2026/)
- [Facebook Privacy Settings 2026 Complete Guide](/privacy-tools-guide/facebook-privacy-settings-2026-complete-guide/)
- [Harden Macos Sequoia Privacy Settings Beyond Default](/privacy-tools-guide/how-to-harden-macos-sequoia-privacy-settings-beyond-default-configuration-complete-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
