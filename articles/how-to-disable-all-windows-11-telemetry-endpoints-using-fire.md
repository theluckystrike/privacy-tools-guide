---
layout: default
title: "How to Disable All Windows 11 Telemetry Endpoints Using."
description: "A comprehensive technical guide for developers and power users to block Windows 11 telemetry using Windows Firewall, including PowerShell scripts and."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-disable-all-windows-11-telemetry-endpoints-using-fire/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Windows 11 collects telemetry data from your system and sends it to Microsoft servers. While Microsoft uses this data to improve the operating system, privacy-conscious developers and power users often want complete control over what leaves their machines. Using Windows Firewall to block telemetry endpoints provides a surgical approach that doesn't require disabling system features or modifying the registry.

This guide covers the comprehensive method of blocking Windows 11 telemetry at the network level using firewall rules.

## Understanding Windows 11 Telemetry

Windows 11 sends data through multiple channels. The main telemetry endpoints include:

- **vortex.data.microsoft.com** - Primary telemetry endpoint
- **settings-win.data.microsoft.com** - Settings and preferences data
- **watson.telemetry.microsoft.com** - Error reporting (Watson)
- **cyd.office.com** - Connected User Experience data
- **sqm.telemetry.microsoft.com** - Software Quality Metrics

The operating system also uses CDN endpoints for content delivery and update-related communications. Blocking these at the firewall level prevents data transmission without affecting system functionality.

## Prerequisites

Before implementing firewall rules, ensure you have:

- Windows 11 Pro, Enterprise, or Education (Home edition can use PowerShell method)
- Administrator privileges
- Understanding that some Windows features may require connectivity

## Method 1: Using PowerShell to Create Firewall Rules

The most efficient method uses PowerShell to create outbound firewall rules that block known telemetry endpoints. This approach works across all Windows 11 editions.

### Basic Telemetry Blocking Script

```powershell
# Run as Administrator
$TelemetryEndpoints = @(
    "vortex.data.microsoft.com",
    "settings-win.data.microsoft.com",
    "watson.telemetry.microsoft.com",
    "cyd.office.com",
    "sqm.telemetry.microsoft.com",
    "telemetry.microsoft.com",
    "oca.telemetry.microsoft.com",
    "watson.microsoft.com",
    "redir.metaservices.microsoft.com",
    "choice.microsoft.com",
    "df.telemetry.microsoft.com",
    "reports.watson.microsoft.com",
    "vortex-sandbox.data.microsoft.com"
)

foreach ($endpoint in $TelemetryEndpoints) {
    $ruleName = "Block-Telemetry-$endpoint"
    
    # Check if rule already exists
    $existingRule = Get-NetFirewallRule -DisplayName $ruleName -ErrorAction SilentlyContinue
    
    if (-not $existingRule) {
        # Create outbound rule to block TCP connections to this endpoint
        New-NetFirewallRule -DisplayName $ruleName `
            -Direction Outbound `
            -Action Block `
            -RemoteAddress $endpoint `
            -Protocol TCP `
            -Enabled True `
            -Profile Any
        
        Write-Host "Created rule: $ruleName"
    } else {
        Write-Host "Rule already exists: $ruleName"
    }
}

Write-Host "Telemetry firewall rules created successfully."
```

This script creates individual outbound rules for each telemetry endpoint. The `-RemoteAddress` parameter accepts domain names, and Windows Firewall resolves these to IP addresses automatically.

### Advanced Blocking with IP Ranges

For more comprehensive coverage, you can block IP ranges that Microsoft uses:

```powershell
# Block Microsoft telemetry IP ranges
$MicrosoftIPRanges = @(
    "104.208.0.0/13",  # Azure
    "40.64.0.0/10",    # Microsoft corporate
    "20.64.0.0/10"     # Microsoft Azure
)

foreach ($ipRange in $MicrosoftIPRanges) {
    $ruleName = "Block-Microsoft-Telemetry-$ipRange"
    
    if (-not (Get-NetFirewallRule -DisplayName $ruleName -ErrorAction SilentlyContinue)) {
        New-NetFirewallRule -DisplayName $ruleName `
            -Direction Outbound `
            -Action Block `
            -RemoteAddress $ipRange `
            -Protocol Any `
            -Enabled True `
            -Profile Any
    }
}
```

Be cautious with IP range blocking, as it may affect legitimate Microsoft services. The endpoint-based approach provides more precise control.

## Method 2: Using Windows Defender Firewall Advanced Security

For users who prefer the graphical interface or need more granular control:

1. Open **Windows Defender Firewall with Advanced Security** (search for it in Start menu)
2. Click **Outbound Rules** in the left sidebar
3. Click **New Rule...** in the right panel
4. Select **Custom** as the rule type
5. Choose **This program or port** based on your preference
6. For program-based rules, specify telemetry-related executables:

```
C:\Windows\System32\CompatTelRunner.exe
C:\Windows\System32\DiagTrack.exe
C:\Windows\WindowsUpdate\*.exe
```

7. Select **Block the connection**
8. Apply to all profiles (Domain, Private, Public)
9. Name the rule appropriately

## Blocking Specific Telemetry Services

Windows 11 runs several services that transmit telemetry. While you can disable these services through other means, blocking their network access provides an additional layer:

### Connected User Experience Service (DiagTrack)

```powershell
# Block DiagTrack.exe network access
$diagPath = "$env:SystemRoot\System32\DiagTrack.exe"

New-NetFirewallRule -DisplayName "Block-DiagTrack" `
    -Direction Outbound `
    -Action Block `
    -Program $diagPath `
    -Enabled True `
    -Profile Any
```

### Compatibility Telemetry Runner

```powershell
# Block CompatTelRunner.exe
$compatPath = "$env:SystemRoot\System32\CompatTelRunner.exe"

New-NetFirewallRule -DisplayName "Block-CompatTelRunner" `
    -Direction Outbound `
    -Action Block `
    -Program $compatPath `
    -Enabled True `
    -Profile Any
```

## Verifying Your Firewall Rules

After implementing the rules, verify they work correctly:

```powershell
# Check all telemetry-related firewall rules
Get-NetFirewallRule | Where-Object { 
    $_.DisplayName -like "*Telemetry*" -or 
    $_.DisplayName -like "*Microsoft*" -and $_.Action -eq "Block" 
} | Select-Object DisplayName, Enabled, Direction, Action | Format-Table
```

### Testing Connectivity

Test whether telemetry endpoints are actually blocked:

```powershell
# Test if telemetry endpoint is reachable
Test-NetConnection -ComputerName "vortex.data.microsoft.com" -Port 443 -WarningAction SilentlyContinue
```

If the firewall rule is working, you should see `TcpTestSucceeded : False` for blocked endpoints.

## Managing Exceptions

Some Windows features require telemetry connectivity. Create exceptions for specific scenarios:

```powershell
# Allow Windows Update while blocking other telemetry
$updateRule = New-NetFirewallRule -DisplayName "Allow-WindowsUpdate" `
    -Direction Outbound `
    -Action Allow `
    -RemoteAddress "windowsupdate.microsoft.com" `
    -Protocol TCP `
    -Enabled True `
    -Profile Any
```

## Removing Rules

To remove all telemetry blocking rules:

```powershell
# Remove all telemetry blocking rules
Get-NetFirewallRule | Where-Object { 
    $_.DisplayName -like "*Telemetry*" -or 
    $_.DisplayName -like "*Block-Microsoft*" 
} | Remove-NetFirewallRule -Confirm:$false

Write-Host "All telemetry blocking rules removed."
```

## Considerations and Trade-offs

Blocking telemetry at the firewall level has implications:

- **Windows Update** may require additional configuration to work properly
- **Error reporting** will no longer function for Microsoft applications
- **Some Settings pages** may show warnings about connectivity issues
- **Cortana** and **search** features may have reduced functionality
- **License activation** and **genuine Windows** checks may be affected

For development environments, this level of blocking is often acceptable. For production systems, consider which features your workflow actually requires.

## Automation Script for Complete Setup

Here's a consolidated script that combines all methods:

```powershell
# Complete Telemetry Firewall Setup
# Run as Administrator

param(
    [switch]$RemoveRules
)

if ($RemoveRules) {
    # Removal logic
    Get-NetFirewallRule | Where-Object { 
        $_.DisplayName -match "Telemetry|Block-Microsoft" 
    } | Remove-NetFirewallRule -Confirm:$false
    Write-Host "All telemetry rules removed."
    exit
}

# Main blocking logic
$endpoints = @(
    "vortex.data.microsoft.com",
    "settings-win.data.microsoft.com",
    "watson.telemetry.microsoft.com",
    "cyd.office.com",
    "sqm.telemetry.microsoft.com"
)

foreach ($endpoint in $endpoints) {
    $ruleName = "Block-Telemetry-$endpoint"
    if (-not (Get-NetFirewallRule -DisplayName $ruleName -ErrorAction SilentlyContinue)) {
        New-NetFirewallRule -DisplayName $ruleName `
            -Direction Outbound `
            -Action Block `
            -RemoteAddress $endpoint `
            -Protocol TCP `
            -Enabled True `
            -Profile Any | Out-Null
    }
}

Write-Host "Telemetry blocking rules applied successfully."
```

Save this as `Set-TelemetryFirewall.ps1` and run with `-RemoveRules` to revert.

## Conclusion

Using Windows Firewall to block telemetry endpoints provides an effective layer of privacy protection without modifying system files or disabling core Windows features. The approach is reversible, granular, and works across all Windows 11 editions.

For developers and power users who need maximum control over their systems, combining firewall rules with other privacy measures (such as disabling telemetry services through Group Policy or Registry) provides comprehensive protection against unwanted data transmission.

Test your configuration after implementation and periodically review firewall rules, as Microsoft may add new endpoints with updates.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
