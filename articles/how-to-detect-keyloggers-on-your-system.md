---
layout: default
title: "How to Detect Keyloggers on Your System"
description: "Find software and hardware keyloggers on Windows, Linux, and macOS using process inspection, network analysis, and USB device auditing"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-detect-keyloggers-on-your-system/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Keyloggers capture every keystroke you type — passwords, private messages, banking credentials. They range from commercial parental-control software to purpose-built malware. Detection requires checking processes, network connections, kernel hooks, and physical hardware.

## Types of Keyloggers

**Software keyloggers** operate at different layers:
- **User-space API hooks**: Hook OS keyboard APIs (Windows `SetWindowsHookEx`, Linux `XRecord`)
- **Kernel drivers**: Load as a kernel module or driver, harder to detect from user space
- **Browser extensions**: Log keystrokes only inside the browser
- **Form grabbers**: Hook browser APIs to steal form submissions

**Hardware keyloggers**:
- USB inline devices that sit between keyboard and computer
- Modified keyboards with embedded chips
- PS/2 inline loggers

## Linux: Detecting Software Keyloggers

### Check for Processes Using Input Devices

```bash
# List processes accessing keyboard input devices
sudo lsof /dev/input/event* 2>/dev/null

# Find processes with elevated privileges checking input
sudo fuser /dev/input/event* 2>/dev/null

# Check which devices are keyboard-type
cat /proc/bus/input/devices | grep -A5 "keyboard\|Keyboard"
```

### Look for X11 Event Grabbers

```bash
# Check for programs using XRecord (X11 keyboard monitoring)
# Install xinput if needed
sudo apt install xinput -y

# List devices and watch for unexpected listeners
xinput list
xinput test-xi2 --root

# Check for XTEST / XRecord usage
xdpyinfo | grep extension
```

### Inspect Kernel Modules

```bash
# List all loaded kernel modules
lsmod | sort

# Look for suspicious or unknown modules
lsmod | grep -v "^Module\|^bluetooth\|^usbcore\|^hid\|^input"

# Get module info
modinfo <suspicious_module_name>

# Check module load history
sudo dmesg | grep -i "module\|keylog\|usbhid"
```

### Audit Running Processes

```bash
# Full process list with open files
ps auxf

# Find processes accessing /dev/input
ls -la /proc/*/fd 2>/dev/null | grep "input/event"

# Check for processes with CAP_SYS_PTRACE (can monitor other processes)
sudo capsh --print
getpcaps $(pidof suspicious_process)
```

### Monitor Network Connections

Keyloggers exfiltrate data. Look for unexpected outbound connections:

```bash
# All established connections
ss -tulpn
netstat -tulpn 2>/dev/null

# Watch for new connections being established
sudo watch -n 2 "ss -tnp | grep ESTABLISHED"

# Capture outbound traffic on the keyboard-use interface
sudo tcpdump -i eth0 -n 'tcp and not port 443 and not port 80' -w /tmp/capture.pcap &
# Type some text, then analyze
sudo tcpdump -r /tmp/capture.pcap
```

## Windows: Detecting Software Keyloggers

### Check Running Processes

```powershell
# Full process list with paths
Get-Process | Select-Object Name, Id, Path | Sort-Object Name

# Find processes without a verified signature
Get-Process | ForEach-Object {
    $sig = Get-AuthenticodeSignature $_.Path -ErrorAction SilentlyContinue
    if ($sig.Status -ne "Valid") {
        "$($_.Name): $($_.Path)"
    }
}
```

### Check for SetWindowsHookEx Hooks

Use Sysinternals Process Explorer or run this PowerShell snippet:

```powershell
# List all global Windows hooks (requires admin)
# Download Autoruns from Sysinternals and check the "Everything" tab
# Or use Process Explorer > View > System Information > Hooks

# Check for DLLs injected into processes
Get-Process | ForEach-Object {
    $proc = $_
    $proc.Modules | Where-Object {
        $_.FileName -notlike "C:\Windows\*" -and
        $_.FileName -notlike "C:\Program Files\*"
    } | ForEach-Object {
        "[$($proc.Name)] $($_.FileName)"
    }
}
```

### Inspect Startup Entries

```powershell
# Check all autorun locations
Get-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce"

# Scheduled tasks
Get-ScheduledTask | Where-Object { $_.State -eq "Ready" } |
  Select-Object TaskName, TaskPath, @{n="Action";e={$_.Actions.Execute}}
```

### Check Network Activity

```powershell
# Active connections with process names
netstat -b -n

# Or with PowerShell
Get-NetTCPConnection -State Established |
  Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort,
    @{n="Process";e={(Get-Process -Id $_.OwningProcess).Name}}
```

## macOS: Detecting Software Keyloggers

```bash
# Check processes with Accessibility access (required for keystroke logging)
sqlite3 "/Library/Application Support/com.apple.TCC/TCC.db" \
  "SELECT client, auth_value FROM access WHERE service='kTCCServiceAccessibility';"

# List Launch Agents and Daemons
ls -la ~/Library/LaunchAgents/
ls -la /Library/LaunchAgents/
ls -la /Library/LaunchDaemons/

# Check for kernel extensions (legacy macOS < 12)
kextstat | grep -v com.apple

# On macOS 12+, check system extensions
systemextensionsctl list

# Network connections with process names
sudo lsof -i -n -P | grep ESTABLISHED
```

## Detecting Hardware Keyloggers

```bash
# List all USB devices
lsusb -v 2>/dev/null | grep -A5 "idVendor\|idProduct\|bcdDevice"

# On Linux, show USB device tree
lsusb -t

# Look for unexpected HID devices
ls /dev/hidraw*
cat /proc/bus/input/devices | grep -B2 -A10 "HID\|Keyboard"

# Check for devices that enumerate as both keyboard AND storage
# (a common hardware keylogger trick)
lsusb | grep -i "mass storage\|flash drive"
dmesg | grep -i "usb\|hid" | tail -30
```

Physically inspect the USB port. A hardware keylogger is a small dongle between the keyboard USB cable and the PC's USB port. It will appear in `lsusb` as an additional device and often has an unfamiliar vendor ID.

## Ongoing Monitoring

Set up auditd to log access to input devices:

```bash
# Watch for opens of keyboard input devices
sudo auditctl -w /dev/input/event0 -p rwa -k keylogger-watch

# View audit events
sudo ausearch -k keylogger-watch --start recent
```

## Related Reading

- [How to Configure UFW Firewall on Ubuntu](/privacy-tools-guide/how-to-configure-ufw-firewall-on-ubuntu/)
- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)
- [How to Monitor File Changes with inotifywait](/privacy-tools-guide/how-to-monitor-file-changes-with-inotifywait/)

- [Complete Guide To Operating System Hardening For Extreme](/privacy-tools-guide/complete-guide-to-operating-system-hardening-for-extreme-pri/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

## Related Articles

- [How to Create Encrypted Partitions with dm-crypt](/privacy-tools-guide/how-to-create-encrypted-partitions-with-dm-crypt/)
- [Verify Your Devices Are Not Compromised: A Complete](/privacy-tools-guide/how-to-verify-your-devices-are-not-compromised-complete-audit/)
- [Privacy-Focused Keyboard Apps for Mobile](/privacy-tools-guide/privacy-focused-keyboard-apps-for-mobile/)
- [How to Check if Your Smart Home Devices Are Compromised](/privacy-tools-guide/how-to-check-if-your-smart-home-devices-are-compromised/)
- [How to Detect and Remove Hidden Tracking Devices on Your](/privacy-tools-guide/how-to-detect-and-remove-hidden-tracking-devices-on-your-car/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
