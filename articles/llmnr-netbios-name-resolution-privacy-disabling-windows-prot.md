---
layout: default
title: "Llmnr Netbios Name Resolution Privacy Disabling Windows"
description: "A technical guide for developers and power users to disable LLMNR and NetBIOS name resolution on Windows, preventing hostname leakage that exposes your"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /llmnr-netbios-name-resolution-privacy-disabling-windows-prot/
categories: [guides]
tags: [privacy-tools-guide, privacy, llm]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Disable LLMNR and NetBIOS on Windows to prevent hostname leakage: run `netsh int ipv4 set global unicastif=enabled` to disable LLMNR, then disable NetBIOS through Network Settings → Advanced → Wins/NetBIOS tab. Alternatively, use Group Policy (gpedit.msc) to disable both at the system level. These protocols broadcast your device name across the network, allowing attackers to discover system information and launch targeted attacks. Disabling them removes this exposure while leaving DNS-based name resolution intact for normal connectivity.

Table of Contents

- [Understanding LLMNR and NetBIOS Name Resolution](#understanding-llmnr-and-netbios-name-resolution)
- [Why These Protocols Create Privacy Risks](#why-these-protocols-create-privacy-risks)
- [Checking Current Status on Your System](#checking-current-status-on-your-system)
- [Disabling LLMNR and NetBIOS on Windows](#disabling-llmnr-and-netbios-on-windows)
- [Verifying Your Configuration](#verifying-your-configuration)
- [Enterprise Deployment Considerations](#enterprise-deployment-considerations)

Understanding LLMNR and NetBIOS Name Resolution

What is LLMNR?

LLMNR (Link-Local Multicast Name Resolution) is a protocol introduced in Windows Vista that allows computers to resolve hostnames without a traditional DNS server. When a Windows machine cannot reach a DNS server, it falls back to LLMNR, sending a multicast query to all devices on the local network asking "Who has this hostname?"

The process works like this - your computer wants to connect to `printer-server.local` but cannot find it in DNS. Instead of failing, Windows broadcasts a multicast message to all devices on the subnet asking which one has that name. Any machine with a matching hostname responds, enabling connectivity.

LLMNR operates on UDP port 5355 and uses IPv4 multicast address 224.0.0.252 or IPv6 multicast group ff02::1.

What is NetBIOS Name Service?

NetBIOS Name Service (NBNS) is an older protocol that predates DNS, originating from IBM's Network Basic Input/Output System. While largely superseded by DNS in modern networks, Windows maintains backward compatibility by keeping NetBIOS enabled by default.

NetBIOS names are 16 characters maximum and historically associated with Windows domain environments. The protocol uses:
- UDP port 137 for name queries
- UDP port 138 for datagram distribution
- TCP port 139 for session services

When NetBIOS is active, your machine responds to name queries from other devices, effectively broadcasting its hostname to anyone who asks.

Why These Protocols Create Privacy Risks

Hostname Leakage

Both protocols expose your device's hostname across the network. Your hostname often contains:
- Your real name (e.g., `john-doe-laptop`)
- Department or team identifiers (e.g., `engineering-workstation-03`)
- Location information (e.g., `nyc-office-mike`)
- Device type or purpose

This information alone can enable social engineering attacks or network mapping.

Man-in-the-Middle Attacks

LLMNR and NetBIOS are susceptible to spoofing attacks. An attacker on the same network can:
1. Monitor for LLMNR or NetBIOS queries
2. Forge responses claiming to be the requested hostname
3. Intercept authentication credentials

This technique, known as LLMNR/NBNS spoofing, has been a staple of penetration testing for over a decade. Responder, a popular open-source tool, automates this attack by listening for name resolution requests and responding with malicious packets.

Network Reconnaissance

Network scanners and monitoring tools can passively collect hostnames without sending any packets. Devices responding to LLMNR or NetBIOS queries automatically reveal their identity, making network mapping trivial for anyone with basic tools.

Checking Current Status on Your System

Before disabling these protocols, verify their current state using PowerShell.

Check LLMNR Status

```powershell
Check if LLMNR is enabled via registry
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name "EnableMulticast" -ErrorAction SilentlyContinue

If the key doesn't exist, LLMNR is enabled by default
A value of 1 means enabled, 0 means disabled
```

Check NetBIOS Status

```powershell
Check NetBIOS over TCP/IP settings for all adapters
Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object {$_.TcpipNetbiosOptions -ne 2} | Select-Object Description, TcpipNetbiosOptions
```

The `TcpipNetbiosOptions` values indicate:
- 0 = Enable NetBIOS via DHCP
- 1 = Enable NetBIOS over TCP/IP
- 2 = Disable NetBIOS over TCP/IP

Disabling LLMNR and NetBIOS on Windows

Method 1 - Group Policy Editor (Windows Pro/Enterprise)

For systems with Group Policy support:

1. Press `Win + R`, type `gpedit.msc`, and press Enter
2. Navigate to: Computer Configuration → Administrative Templates → Network → DNS Client
3. Enable "Turn off multicast name resolution" and set it to Disabled
4. This disables LLMNR system-wide

For NetBIOS, you'll need to modify network adapter settings or use a startup script.

Method 2 - Registry Modification

Create a `.reg` file or use PowerShell to modify the registry directly.

Disable LLMNR via Registry

```powershell
Create registry key if it doesn't exist and disable LLMNR
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
if (!(Test-Path $regPath)) {
    New-Item -Path $regPath -Force | Out-Null
}
Set-ItemProperty -Path $regPath -Name "EnableMulticast" -Value 0 -Type DWord
```

Disable NetBIOS over TCP/IP via Registry

```powershell
Get all network adapters and disable NetBIOS
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object {$_.IPEnabled -eq $true}

foreach ($adapter in $adapters) {
    # 2 = Disable NetBIOS over TCP/IP
    $adapter.SetTcpipNetbios(2) | Out-Null
}
```

Method 3 - PowerShell Script for Complete Disabling

This script disables both protocols:

```powershell
Disable-LLMNR-NetBIOS.ps1
Run as Administrator

Write-Host "Disabling LLMNR..." -ForegroundColor Cyan

Disable LLMNR via registry
$llmnrPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
if (!(Test-Path $llmnrPath)) {
    New-Item -Path $llmnrPath -Force | Out-Null
}
Set-ItemProperty -Path $llmnrPath -Name "EnableMulticast" -Value 0 -Type DWord

Write-Host "Disabling NetBIOS over TCP/IP..." -ForegroundColor Cyan

Disable NetBIOS on all enabled adapters
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object {$_.IPEnabled -eq $true}

foreach ($adapter in $adapters) {
    $result = $adapter.SetTcpipNetbios(2)
    if ($result.ReturnValue -eq 0) {
        Write-Host "Disabled NetBIOS on: $($adapter.Description)" -ForegroundColor Green
    }
}

Write-Host "`nVerification - Current LLMNR status:" -ForegroundColor Yellow
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name "EnableMulticast" -ErrorAction SilentlyContinue

Write-Host "`nVerification - NetBIOS status:" -ForegroundColor Yellow
Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object {$_.IPEnabled -eq $true} |
    Select-Object Description, TcpipNetbiosOptions

Write-Host "`nDone. A restart may be required for changes to take full effect." -ForegroundColor Green
```

Save this as `Disable-LLMNR-NetBIOS.ps1` and execute with administrator privileges.

Method 4 - Disable via PowerShell Direct Commands

For quick one-liner execution:

```powershell
Disable LLMNR
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name "EnableMulticast" -Value 0 -Type DWord -Force

Disable NetBIOS on all adapters
Get-NetAdapter | ForEach-Object {
    Set-NetAdapterIPConfiguration -InterfaceIndex $_.ifIndex -DisableNetBIOSOverTcpIP
}
```

Verifying Your Configuration

After applying changes, verify that name resolution queries are no longer being broadcast:

Capture Network Traffic

Use Wireshark or PowerShell to monitor network activity:

```powershell
Monitor for LLMNR traffic (requires admin)
Get-NetUDPEndpoint -LocalPort 5355 -ErrorAction SilentlyContinue

Monitor for NetBIOS name queries
Get-NetUDPEndpoint -LocalPort 137 -ErrorAction SilentlyContinue
```

If these commands return no output, the respective services are not listening on those ports.

Test Name Resolution Behavior

```powershell
Attempt to resolve a non-existent hostname
Without LLMNR/NetBIOS, this should fail immediately
Resolve-DnsName "nonexistent-hostname-12345" -QuickTimeout -ErrorAction SilentlyContinue
```

Enterprise Deployment Considerations

For organizations managing multiple machines:

Using Group Policy

Create a Group Policy Object (GPO) with the registry settings:
- Path: `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient`
- Value: `EnableMulticast` = 0 (DWORD)

Using Microsoft Intune

Deploy a configuration profile with:
- Endpoint protection → Windows Defender Firewall → Block LLMNR
- Or use a PowerShell script for NetBIOS disabling

Impact Assessment

Before deploying organization-wide, verify that:
- Legacy systems do not rely on NetBIOS for name resolution
- Applications using custom multicast discovery are identified
- Printers or IoT devices using NetBIOS are documented

Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

Can I trust these tools with sensitive data?

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

Related Articles

- [Windows 11 Privacy Settings: How to Disable Telemetry](/windows-11-privacy-settings-disable-telemetry/)
- [Windows 11 Cortana Disable Privacy Guide](/windows-11-cortana-disable-privacy-guide/)
- [Windows 10 Privacy Settings Complete Checklist](/windows-10-privacy-settings-complete-checklist/)
- [Windows Group Policy Privacy Settings Guide](/windows-group-policy-privacy-settings-guide/)
- [Chromebook Privacy Settings for Students 2026](/chromebook-privacy-settings-for-students-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
