---
permalink: /how-to-verify-your-devices-are-not-compromised-complete-audit/
---
layout: default
title: "Verify Your Devices Are Not Compromised: A Complete"
description: "A practical guide for developers and power users to audit device security. Learn to identify signs of compromise through process analysis, network monitoring"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-verify-your-devices-are-not-compromised-complete-audit/
categories: [security, guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Create baseline snapshots of running processes, installed packages, open network connections, and scheduled tasks during a known-clean state. Periodically compare current snapshots against baselines using diff tools to identify new processes, ports, or packages. Monitor system logs for suspicious authentication attempts, privilege escalations, or file modifications. Check for unexpected cron jobs, systemd timers, startup hooks, and browser extensions. Use integrity checkers like `aide` or `tripwire` to alert on unauthorized file modifications to system binaries.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Starting Point: Document Your Baseline

Before detecting anomalies, establish what "normal" looks like for your system. Create a baseline snapshot during a known-clean state:

```bash
# Linux/macOS: Export running processes
ps aux > ~/baseline_processes_$(date +%Y%m%d).txt

# Linux: Record installed packages
dpkg -l > ~/baseline_packages_$(date +%Y%m%d).txt  # Debian/Ubuntu
rpm -qa > ~/baseline_packages_$(date +%Y%m%d).txt  # RHEL/CentOS

# macOS: List installed applications
ls /Applications > ~/baseline_apps_$(date +%Y%m%d).txt
```

Store these baselines securely—preferably offline or in encrypted storage. Compare current system state against these snapshots during audits.

### Step 2: Process Analysis

Unfamiliar processes often indicate compromise. Start by listing all running processes and checking for suspicious entries.

### Linux Inspection

```bash
# View all processes with detailed information
ps auxf

# Check for processes running on unusual ports
sudo netstat -tulpn | grep LISTEN

# Find processes by name or pattern
ps -ef | grep -i "crypto\|miner\|nc \|netcat"
```

Suspicious indicators include:
- Processes with hidden command arguments (showing as `[kworker]` or similar)
- Unknown processes consuming high CPU or network bandwidth
- Processes spawning multiple children that disappear quickly

### macOS Inspection

```bash
# List all running processes
ps -aux

# Check for kext (kernel extension) loading
kextstat

# Monitor new processes in real-time
sudo dtrace -n 'proc:::exec-success { printf("%s %s\n", execname, curpsinfo->pr_psargs); }'
```

### Windows PowerShell

```powershell
# List processes with CPU and memory usage
Get-Process | Sort-Object CPU -Descending | Select-Object -First 20

# Check for suspicious services
Get-Service | Where-Object {$_.Status -eq 'Running'}

# Search for known malware process names
Get-Process | Where-Object {$_.Name -match 'cryptominer|coinhive|minerd'}
```

### Step 3: Network Connection Audit

Compromised devices often maintain unauthorized network connections for command-and-control (C2) communication or data exfiltration.

### Linux/macOS Network Analysis

```bash
# Show all network connections with associated processes
sudo lsof -i -P -n

# Check established connections
sudo netstat -tn | grep ESTABLISHED

# Monitor DNS queries (shows domains being resolved)
sudo tcpdump -i any -n 'port 53' -c 50

# List listening ports
sudo ss -tulpn
```

Review connections against your expected network traffic. Unexpected external IPs, especially connections to known malicious domains, warrant immediate investigation.

### Windows Network Analysis

```powershell
# Check active network connections
Get-NetTCPConnection | Where-Object {$_.State -eq 'Established'}

# Find processes with network connections
Get-NetTCPConnection | Select-Object OwningProcess, LocalAddress, RemoteAddress | ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Process = $proc.ProcessName
        Local = $_.LocalAddress
        Remote = $_.RemoteAddress
    }
}
```

### Step 4: System Integrity Verification

Verify that critical system files remain unmodified. Rootkits and malware often replace core system binaries.

### Linux Integrity Checks

```bash
# Install and run rkhunter
sudo apt-get install rkhunter  # Debian/Ubuntu
sudo rkhunter --checkall

# Use chkrootkit
sudo apt-get install chkrootkit
sudo chkrootkit

# Verify package integrity (Debian)
dpkg --verify

# Compare file hashes against known-good values
sha256sum /bin/ls /bin/ps /usr/bin/top
```

The `aide` (Advanced Intrusion Detection Environment) package provides file integrity monitoring:

```bash
sudo apt-get install aide
sudo aideinit
sudo aide --check
```

### macOS Gatekeeper Verification

```bash
# Verify system file signatures
spctl --master-enable
spctl --status

# Check for unsigned system modifications
sudo codesign -vvv /System/Library/Extensions/*.kext

# Review system log for code signature failures
log show --predicate 'eventMessage contains "code signature"' --last 1h
```

### Step 5: Authentication and Access Review

Compromised credentials enable unauthorized access. Audit user accounts and authentication logs.

### Linux User Audit

```bash
# List all users with login access
cat /etc/passwd | grep -v '/nologin\|/false'

# Check for new or unexpected users
sudo lastlog

# Review failed login attempts
sudo lastb | head -50

# Check sudo access
getent group sudo
```

### SSH Key Audit

```bash
# Review authorized_keys
cat ~/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys

# Check for SSH agent forwarding abuse
ssh-add -l
```

### macOS Authentication Audit

```bash
# Review login history
last

# Check for new keyboard layouts or input sources
defaults read com.apple.HIToolbox | grep -i "keyboard"

# Audit login items
ls -la ~/Library/Application\ Support/com.apple.backgroundtaskmanagementagent/
```

### Step 6: Log Analysis

System and application logs reveal historical anomalies. Focus on authentication, privilege escalation, and error events.

### Linux Log Review

```bash
# Authentication failures
sudo grep -i "failed password\|authentication failure" /var/log/auth.log | tail -50

# Sudo command usage
sudo grep -i "COMMAND=" /var/log/auth.log | tail -20

# Kernel messages (may show hardware/driver issues or exploits)
dmesg | tail -100
```

### macOS Log Analysis

```bash
# Use log command for structured filtering
log show --predicate 'eventMessage contains "authentication"' --last 24h

# Check firewall events
log show --predicate 'eventMessage contains "firewall"' --last 24h

# Review install and removal events
log show --predicate 'eventMessage contains "install"' --last 7d | grep -i "pkg\|app"
```

### Step 7: Storage and Persistence Check

Malware often establishes persistence through startup items, cron jobs, or scheduled tasks.

### Linux Persistence Mechanisms

```bash
# Check cron jobs (system and user)
sudo crontab -l
ls -la /etc/cron.d/
cat /etc/crontab

# Review systemd services
systemctl list-units --type=service --state=running

# Check init.d scripts
ls -la /etc/init.d/

# Examine startup scripts
ls -la /etc/rc.local
```

### macOS Launch Items

```bash
# List launch agents and daemons
ls -la ~/Library/LaunchAgents/
ls -la /Library/LaunchAgents/
ls -la /Library/LaunchDaemons/

# Check login items
defaults read com.apple.loginwindow LoginItems
```

### Windows Persistence Check

```powershell
# List scheduled tasks
Get-ScheduledTask | Where-Object {$_.State -eq 'Ready'}

# Check registry Run keys
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'
Get-ItemProperty -Path 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'

# List services
Get-Service | Select-Object Name, Status, StartType
```

### Step 8: Practical Audit Workflow

Execute a systematic audit using this sequence:

1. **Snapshot current state** — Export process list, network connections, and user sessions before making changes
2. **Network sweep** — Identify unexpected connections or listening ports
3. **Process review** — Match running processes against your baseline
4. **Log mining** — Search for authentication anomalies in the past 7 days
5. **Persistence check** — Review startup items, cron jobs, and services
6. **Integrity verification** — Run rkhunter, chkrootkit, or aide
7. **Document findings** — Record all anomalies for further investigation

## When to Escalate

Some findings require immediate escalation beyond personal investigation:

- Root-level compromises indicated by kernel rootkits
- Unauthorized access from unknown IP addresses
- Modification of system binaries or kernel modules
- Evidence of data exfiltration in network logs
- Unknown user accounts with administrative privileges

In these cases, isolate the affected system from the network, preserve forensic evidence, and consider engaging security professionals.

### Step 9: Build Audit Habits

Perform device audits regularly—monthly for development machines handling sensitive work, quarterly for general systems. Automate baseline comparisons using tools like AIDE or OSSEC for continuous monitoring. Document your expected network topology and compare actual connections against it during each audit.

The goal is not paranoia but disciplined verification. Regular audits catch compromises early, limiting damage and enabling faster recovery.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Verify Your Devices Are Not Compromised](/privacy-tools-guide/how-to-verify-your-devices-are-not-compromised-complete-audi/)
- [How to Check if Your Smart Home Devices Are Compromised](/privacy-tools-guide/how-to-check-if-your-smart-home-devices-are-compromised/)
- [Cloud Storage Security Breach History: Compromised.](/privacy-tools-guide/cloud-storage-security-breach-history-compromised-services-t/)
- [How to Detect if Your Email Is Compromised](/privacy-tools-guide/detect-email-compromise-guide)
- [Communicate with Lawyer Privately When Device is Compromised](/privacy-tools-guide/how-to-communicate-with-lawyer-privately-when-device-compromised/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
