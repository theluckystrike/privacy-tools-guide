---
layout: default
title: "How to Harden Windows 10 and 11 Privacy Settings"
description: "Disable Windows telemetry, advertising IDs, and data collection using Group Policy, registry tweaks, and PowerShell scripts on Windows 10 and 11"
date: 2026-03-22
author: theluckystrike
permalink: /harden-windows-11-privacy-settings/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}


Windows collects diagnostic data, browsing habits, clipboard content, location data, and usage telemetry by default. Some collection can't be disabled through the UI — it requires Group Policy, registry edits, or PowerShell. This guide covers what to disable and how to verify it's actually off.

---

## What Windows Collects by Default

- **Diagnostic data**: App crashes, driver failures, OS usage statistics
- **Inking & typing personalization**: Everything you type (to improve autocomplete and handwriting recognition)
- **Advertising ID**: Per-user ID for targeted ads across Microsoft services and third-party apps
- **Location history**: GPS-level position data stored for 24 hours and shared with apps
- **Search history**: Bing searches from the Start menu and taskbar
- **Activity history**: Apps used, files opened (synced to Timeline)
- **Clipboard history**: Contents of your clipboard (on Windows 11, synced to cloud)
- **Wi-Fi Sense**: Shares Wi-Fi passwords with contacts (Windows 10)

---

## Quick Win: Privacy Settings UI

Start here before the advanced steps:

```
Settings → Privacy & security → General:
  - Let apps show personalized ads: OFF
  - Let Windows improve Start and search results: OFF
  - Show suggested content in Settings: OFF
  - Let apps use advertising ID: OFF

Settings → Privacy & security → Diagnostics & feedback:
  - Diagnostic data: Required only (minimum)
  - Improve inking & typing: OFF
  - Tailored experiences: OFF
  - View diagnostic data → Delete diagnostic data

Settings → Privacy & security → Activity history:
  - Store my activity history: OFF
  - Send my activity history to Microsoft: OFF
  - Clear activity history → Clear

Settings → Privacy & security → Location:
  - Location services: OFF (or per-app)

Settings → Privacy & security → Search permissions:
  - Safe Search: OFF
  - Search history → Clear search history
  - Show search highlights: OFF
```

---

## Disable Telemetry via Group Policy (Pro/Enterprise)

Group Policy Editor (`gpedit.msc`) is available on Windows Pro and Enterprise:

```
Computer Configuration → Administrative Templates → Windows Components → Data Collection and Preview Builds

"Allow Diagnostic Data" → Enabled → Set to: Diagnostic data off (not recommended) or Security

Computer Configuration → Administrative Templates → Windows Components → Search
"Allow Cortana" → Disabled
"Do not allow web search" → Enabled
"Don't search the web or display web results in Search" → Enabled

Computer Configuration → Administrative Templates → Windows Components → Windows Update
"Do not include drivers with Windows Updates" → Enabled
(prevents automatic driver installation that may include telemetry)
```

---

## Registry Tweaks (All Windows Editions)

For Windows Home (no Group Policy Editor), use the Registry Editor (`regedit`):

```bat
:: Run as Administrator

:: Disable telemetry (0 = Security level, 1 = Basic, 3 = Full)
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f

:: Disable Cortana
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowCortana /t REG_DWORD /d 0 /f

:: Disable web search in Start
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v DisableWebSearch /t REG_DWORD /d 1 /f

:: Disable advertising ID
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\AdvertisingInfo" /v DisabledByGroupPolicy /t REG_DWORD /d 1 /f

:: Disable location services
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\LocationAndSensors" /v DisableLocation /t REG_DWORD /d 1 /f

:: Disable feedback frequency
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v DoNotShowFeedbackNotifications /t REG_DWORD /d 1 /f
```

---

## PowerShell Hardening Script

```powershell
# Run PowerShell as Administrator

# Disable telemetry services
$services = @(
    "DiagTrack",           # Connected User Experiences and Telemetry
    "dmwappushservice",    # WAP Push Message Routing Service
    "SysMain",             # Superfetch (sends usage data)
    "WerSvc",              # Windows Error Reporting
    "WSearch"              # Windows Search (if not needed)
)

foreach ($svc in $services) {
    Stop-Service -Name $svc -Force -ErrorAction SilentlyContinue
    Set-Service -Name $svc -StartupType Disabled -ErrorAction SilentlyContinue
    Write-Host "Disabled: $svc"
}

# Disable scheduled telemetry tasks
$tasks = @(
    "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser",
    "\Microsoft\Windows\Application Experience\ProgramDataUpdater",
    "\Microsoft\Windows\Autochk\Proxy",
    "\Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
    "\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",
    "\Microsoft\Windows\DiskDiagnostic\Microsoft-Windows-DiskDiagnosticDataCollector",
    "\Microsoft\Windows\Feedback\Siuf\DmClient",
    "\Microsoft\Windows\Maintenance\WinSAT",
    "\Microsoft\Windows\Windows Error Reporting\QueueReporting"
)

foreach ($task in $tasks) {
    Disable-ScheduledTask -TaskPath (Split-Path $task) -TaskName (Split-Path $task -Leaf) -ErrorAction SilentlyContinue
    Write-Host "Disabled task: $task"
}

# Disable Windows Defender telemetry (keeps AV protection, stops data sharing)
Set-MpPreference -MAPSReporting Disabled
Set-MpPreference -SubmitSamplesConsent Never

Write-Host "`nDone. Reboot recommended."
```

---

## Block Microsoft Telemetry Domains (Hosts File)

Even with services disabled, Windows may use scheduled tasks or secondary processes to report telemetry. Block at the DNS/hosts level:

```powershell
# PowerShell: append telemetry domains to hosts file
$hostsPath = "$env:SystemRoot\System32\drivers\etc\hosts"
$domains = @(
    "vortex.data.microsoft.com",
    "vortex-win.data.microsoft.com",
    "telecommand.telemetry.microsoft.com",
    "telecommand.telemetry.microsoft.com.nsatc.net",
    "oca.telemetry.microsoft.com",
    "sqm.telemetry.microsoft.com",
    "watson.telemetry.microsoft.com",
    "redir.metaservices.microsoft.com",
    "settings-sandbox.data.microsoft.com",
    "reports.wes.df.telemetry.microsoft.com",
    "services.wes.df.telemetry.microsoft.com",
    "sqm.df.telemetry.microsoft.com",
    "telemetry.microsoft.com",
    "activity.windows.com",
    "browser.pipe.aria.microsoft.com"
)

foreach ($d in $domains) {
    $line = "0.0.0.0 $d"
    if (-not (Select-String -Path $hostsPath -Pattern [regex]::Escape($d) -Quiet)) {
        Add-Content -Path $hostsPath -Value $line
        Write-Host "Blocked: $d"
    }
}
```

---

## Windows 11 Specific: Disable Recall and AI Features

Windows 11 24H2 introduced Copilot+ features including Recall (takes screenshots every few seconds):

```
Settings → Privacy & security → Recall & snapshots:
  - Save snapshots: OFF

Settings → Personalization → Device usage → toggle off all usage types
Settings → System → Clipboard → Clipboard history: OFF
Settings → System → Clipboard → Sync across devices: OFF

# Via registry (for automation):
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\screenCapture" /v Value /t REG_SZ /d Deny /f
```

---

## Verify What's Phoning Home

```powershell
# Monitor network connections from Windows processes
netstat -b 5 | findstr /i "microsoft"

# Or use Process Monitor from Sysinternals (free):
# Download: live.sysinternals.com/Procmon.exe
# Filter: Operation = TCP Connect, then filter by address contains "microsoft"
```

Tools like Glasswire (paid) or the free Wireshark can capture and filter traffic:

```bash
# Windows Subsystem for Linux: use tshark (terminal Wireshark)
tshark -i "Wi-Fi" -f "host microsoft.com or host bing.com" -Y "tcp" -T fields -e ip.dst -e http.host
```

---

## What You Can't Fully Disable

Some telemetry is hardcoded into Windows Update and cannot be disabled without breaking update functionality:

- Windows Update communicates metadata about your installed software to check compatibility
- Microsoft Defender sample submission (can reduce but not fully disable without compromising protection)
- Windows Hello/Biometric data (stored locally but requires cloud authentication for some enterprise features)

For users needing maximum privacy, consider: Windows LTSC edition (Long Term Servicing Channel) — stripped of most consumer telemetry by design, used in enterprise and kiosk deployments.

---

## Related Reading

- [Windows Privacy Tools Open Source 2026](/privacy-tools-guide/windows-privacy-tools-open-source-2026/)
- [Android Privacy Best Practices 2026](/privacy-tools-guide/android-privacy-best-practices-2026/)
- [Block Web Trackers Without Extensions](/privacy-tools-guide/block-web-trackers-without-extensions/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)


## Frequently Asked Questions


**How long does it take to harden windows 10 and 11 privacy settings?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.


**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.


**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.


**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.


**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.


{% endraw %}
