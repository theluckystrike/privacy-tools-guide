---

layout: default
title: "How to Verify Your Devices Are Not Compromised: A."
description: "A practical guide for developers and power users to audit device security, detect compromises, and verify system integrity through command-line tools."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-verify-your-devices-are-not-compromised-complete-audi/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

When you suspect that your device may have been compromised, a systematic audit helps you confirm whether your system is secure or if unauthorized access has occurred. This guide provides a practical framework for verifying device integrity using command-line tools, forensic techniques, and detection methods that work across macOS, Linux, and Windows systems.

## Why Device Audits Matter

Developers and power users store sensitive data, API keys, SSH credentials, and personal information on their machines. A compromised device can lead to data theft, unauthorized code commits, or lateral movement across your digital infrastructure. Regular security audits help you detect intrusions early and maintain confidence in your system's integrity.

The audit process involves checking running processes, examining network connections, reviewing system logs, verifying file integrity, and analyzing authentication events. Each category provides different signals that together paint a picture of your device's security posture.

## Checking Running Processes

Unwanted or suspicious processes often indicate malware, backdoors, or unauthorized monitoring tools. On Linux and macOS, use the `ps` command with aux flags to see all running processes:

```bash
ps aux | head -20
```

For a more detailed view that updates in real-time:

```bash
top -c
```

On Windows, open PowerShell and run:

```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 20
```

Look for processes with unusual names, high CPU usage from unknown applications, or processes running under unexpected user accounts. Pay special attention to processes named similarly to system utilities but located in unexpected directories.

To find processes listening on network ports:

```bash
# Linux/macOS
sudo lsof -i -P -n

# Windows
netstat -ano | findstr "LISTENING"
```

Compare the output against your expectations. Any listening service you did not install warrants investigation.

## Examining Network Connections

Malware often communicates with command-and-control servers or exfiltrates data over network connections. Check active network connections and routing tables:

```bash
# View all active connections
netstat -tuln

# Check established connections
netstat -tn | grep ESTABLISHED

# Display routing table
route -n
```

For deeper analysis, tools like `nmap` allow you to scan your own device:

```bash
nmap -sV localhost
nmap -sT localhost
```

If you discover connections to IP addresses you do not recognize, investigate those IPs using threat intelligence databases. Tools like `curl ipinfo.io/<IP>` or `whois <IP>` provide context about the connection origin.

## Analyzing Authentication Logs

Reviewing authentication logs reveals failed login attempts, successful logins from unexpected locations, and session anomalies.

On Linux systems:

```bash
# View recent authentication events
last -20
sudo journalctl -u sshd | tail -50
sudo cat /var/log/auth.log | tail -100
```

On macOS:

```bash
# Use the log command to query authentication events
log show --predicate 'eventMessage contains "sshd" OR eventMessage contains "login"' --last 24h
```

On Windows, open Event Viewer and navigate to Security logs, or use PowerShell:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddDays(-7)} | Where-Object {$_.Message -match 'logon|Logon'} | Select-Object -First 50
```

Look for multiple failed authentication attempts, successful logins at unusual hours, or logins from unfamiliar IP addresses.

## Verifying File Integrity

Compromised systems often have modified system binaries, suspicious files in temporary directories, or unauthorized cron jobs and launch agents.

Check for recently modified system files:

```bash
# Linux: Find files modified in the last 7 days in system directories
sudo find /usr/bin /usr/sbin /bin /sbin -mtime -7 -type f

# macOS: Similar approach
sudo find /bin /usr/bin /usr/sbin /System/Library -mtime -7 -type f
```

Review startup items and scheduled tasks:

```bash
# Linux: Check cron jobs
crontab -l
sudo cat /etc/crontab
sudo ls -la /etc/cron.d/

# macOS: Check launch agents
ls ~/Library/LaunchAgents/
sudo ls /Library/LaunchAgents/
sudo ls /Library/LaunchDaemons/

# Windows: Check scheduled tasks
schtasks /query /fo LIST /v
```

On Linux, the `aide` tool provides file integrity monitoring by maintaining a database of file hashes and detecting changes:

```bash
# Initialize aide database
sudo aideinit

# Check for changes
sudo aide --check
```

## Checking for Rootkits and Malware

Rootkits hide malicious activity by intercepting system calls and modifying kernel structures. Specialized tools detect these threats.

On Linux, use `rkhunter` or `chkrootkit`:

```bash
# Install and run chkrootkit
sudo apt-get install chkrootkit
sudo chkrootkit

# Install and run rkhunter
sudo apt-get install rkhunter
sudo rkhunter --check
```

On macOS, the open-source `knockknock` tool enumerates persistent startup items:

```bash
# Install via Homebrew
brew install knockknock
knockknock --output-format text
```

For malware scanning, consider ClamAV:

```bash
# Install ClamAV
sudo apt-get install clamav

# Update signatures
sudo freshclam

# Scan home directory
clamscan -r ~/ --exclude-dir=/home/\.cache
```

## System Integrity Verification

macOS and iOS users can verify system integrity using built-in tools. On macOS, check for Gatekeeper and System Integrity Protection status:

```bash
# Verify Gatekeeper status
spctl --status

# Check SIP status
csrutil status
```

On iOS, the jailbreak community has developed tools to detect compromise, though these require careful interpretation as some jailbreaks now hide from detection.

## Practical Audit Checklist

Perform these steps in order for an audit:

1. **Document baseline**: Before issues arise, record your normal process list, network connections, and startup items
2. **Review running processes**: Identify unknown or suspicious processes
3. **Examine network connections**: Look for unexpected outbound connections
4. **Analyze authentication logs**: Check for unauthorized access attempts
5. **Verify file integrity**: Confirm system files remain unmodified
6. **Scan for malware**: Use specialized detection tools
7. **Review access tokens**: Check active sessions and authorized devices

## Responding to Findings

If your audit reveals compromise indicators:

- Disconnect from the network immediately
- Do not log into sensitive accounts from the compromised device
- Backup essential data using a verified clean environment
- Consider reformatting and reinstalling the operating system for severe infections
- Rotate all credentials, API keys, and passwords that were accessible from the device
- Enable two-factor authentication on all accounts

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
