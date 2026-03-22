---
layout: default
title: "Setting Up ClamAV Antivirus on Linux"
description: "Install and configure ClamAV on Linux with automated signature updates, on-access scanning via clamonacc, email scanning, and scheduled scan reports"
date: 2026-03-22
author: theluckystrike
permalink: /clamav-antivirus-linux-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Setting Up ClamAV Antivirus on Linux

ClamAV is the standard open-source antivirus for Linux. It's particularly useful for mail servers (scanning attachments), shared storage, and systems that handle files from Windows machines. On a Linux desktop, the main threat model is files you receive from others — email attachments, downloaded archives, USB drives from other people.

This guide covers a complete ClamAV deployment: installation across major distributions, signature update automation, on-demand and on-access scanning, scheduled weekly scans, quarantine management, and mail server integration with Postfix.
---

## Key Takeaways

- **`LocalSocketMode 666` is needed**: so non-root users and mail system daemons can communicate with clamd over the Unix socket.
- **If you're running in**: a more locked-down environment, use `660` and add the appropriate users to the `clamav` group instead.
- **It's particularly useful for**: mail servers (scanning attachments), shared storage, and systems that handle files from Windows machines.
- **Use `clamdscan` (which delegates**: to the already-running daemon) for bulk or frequent scans.
- **The `--remove` flag permanently deletes infected files without confirmation**: use `--move=/path/to/quarantine` if you want to review before destroying.
- **`OnAccessExcludeUname clamav` and `OnAccessExcludeUname**: root` prevent `clamonacc` from scanning files accessed by the ClamAV process itself or root, which would cause recursive scan loops.

## Install ClamAV

```bash
# Debian/Ubuntu
sudo apt install clamav clamav-daemon

# RHEL/Fedora/CentOS
sudo dnf install clamav clamav-update clamav-scanner-systemd

# Arch Linux
sudo pacman -S clamav

clamscan --version
```

On Debian-based systems, `clamav-daemon` installs the `clamd` background process. On RHEL/Fedora, `clamav-scanner-systemd` provides the systemd unit files. After installation, confirm you see a version string like `ClamAV 1.0.x/26xxx/` — that trailing number is the current signature database revision.

---

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Update Virus Definitions

ClamAV's detection effectiveness depends entirely on current signatures. The `freshclam` utility handles downloading updates from the ClamAV mirror network.

```bash
# Stop the daemon before manual update
sudo systemctl stop clamav-freshclam

# Download latest definitions
sudo freshclam

# Check database files
ls -lh /var/lib/clamav/
# main.cvd, daily.cvd, bytecode.cvd should be recent

# Enable automatic updates
sudo systemctl enable --now clamav-freshclam
```

The three database files each serve a specific purpose. `main.cvd` is the core signature database updated infrequently. `daily.cvd` contains signatures added since the last main release and is updated multiple times daily. `bytecode.cvd` contains bytecode signatures that allow ClamAV to run sandboxed code to detect polymorphic malware — this is what catches threats that change their binary footprint to evade static signature matching.

**Configure freshclam:**

```bash
sudo nano /etc/clamav/freshclam.conf

# Key settings:
Checks 24                              # Check 24 times per day
LogFile /var/log/clamav/freshclam.log
NotifyClamd /etc/clamav/clamd.conf
```

`NotifyClamd` is important: after freshclam downloads new definitions, it signals the running `clamd` daemon to reload them without a restart. Without this, your daemon continues using stale signatures until you restart it manually.

---

### Step 2: Basic Scanning

```bash
# Scan a single file
clamscan /path/to/suspicious-file.pdf

# Scan a directory recursively
clamscan -r /home/user/Downloads/

# Scan with detailed output and remove infected files
clamscan -r --infected --remove /home/user/Downloads/

# Scan and log results
clamscan -r --log=/var/log/clamav/scan-$(date +%Y%m%d).log /home/user/

# Scan a USB drive
clamscan -r /media/usb/ --infected

# Scan archives
clamscan --scan-archive=yes /path/to/archive.zip
```

`clamscan` launches a new process for each scan session, loading the full signature database each time. For a single suspicious file this is fine, but scanning an entire home directory this way is slow — loading 300+ MB of signatures per invocation adds up. Use `clamdscan` (which delegates to the already-running daemon) for bulk or frequent scans.

The `--infected` flag suppresses output for clean files and shows only detections, which makes logs readable when scanning large directories. The `--remove` flag permanently deletes infected files without confirmation — use `--move=/path/to/quarantine` if you want to review before destroying.

---

### Step 3: Configure the ClamAV Daemon (clamd)

The daemon loads virus definitions into memory and accepts scan requests — much faster than clamscan for each file.

```bash
sudo nano /etc/clamav/clamd.conf
```

```ini
LocalSocket /var/run/clamav/clamd.ctl
LocalSocketMode 666
LogFile /var/log/clamav/clamav.log
LogTime yes

MaxScanSize 100M
MaxFileSize 25M
MaxRecursion 10

ScanArchives yes
ScanPDF yes
ScanHTML yes
ScanELF yes
```

```bash
sudo systemctl enable --now clamav-daemon

# Verify daemon is running
echo "PING" | nc -U /var/run/clamav/clamd.ctl
# Response: PONG

# Scan using daemon (faster)
clamdscan -r /home/user/Downloads/
```

`MaxScanSize` controls the maximum amount of data extracted from archives before ClamAV gives up scanning that archive. `MaxFileSize` limits individual files — files larger than this are skipped with a warning logged. Setting these too high risks resource exhaustion from archive bombs (maliciously crafted zip files that expand to gigabytes).

`LocalSocketMode 666` is needed so non-root users and mail system daemons can communicate with clamd over the Unix socket. If you're running in a more locked-down environment, use `660` and add the appropriate users to the `clamav` group instead.

---

### Step 4: On-Access Scanning (Real-Time)

`clamonacc` provides real-time scanning of files as they're accessed:

```bash
# Add to clamd.conf:
OnAccessIncludePath /home
OnAccessExcludeUname clamav
OnAccessExcludeUname root
MaxThreads 2
OnAccessPrevention yes
OnAccessExtraScanning yes
```

```ini
# /etc/systemd/system/clamav-clamonacc.service
[Unit]
Description=ClamAV On-Access Scanner
Requires=clamav-daemon.service
After=clamav-daemon.service

[Service]
Type=simple
ExecStart=/usr/sbin/clamonacc -F --log=/var/log/clamav/clamonacc.log \
  --move=/var/lib/clamav/quarantine
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now clamav-clamonacc
tail -f /var/log/clamav/clamonacc.log
```

`OnAccessPrevention yes` blocks access to a file until scanning completes. With this disabled, `clamonacc` scans files but does not prevent them from being executed — it alerts only. On desktop systems where you regularly download and immediately run scripts or binaries, enabling prevention makes sense. On servers with high I/O throughput, measure the performance impact before enabling it.

`OnAccessExcludeUname clamav` and `OnAccessExcludeUname root` prevent `clamonacc` from scanning files accessed by the ClamAV process itself or root, which would cause recursive scan loops.

---

### Step 5: Scheduled Scans

```bash
sudo nano /etc/cron.weekly/clamav-scan
```

```bash
#!/bin/bash
LOG=/var/log/clamav/weekly-scan-$(date +%Y%m%d).log
QUARANTINE=/var/lib/clamav/quarantine

mkdir -p "$QUARANTINE"

clamscan -r --infected --move="$QUARANTINE" \
    --log="$LOG" \
    --exclude-dir=/proc \
    --exclude-dir=/sys \
    --exclude-dir=/dev \
    /home /tmp /var/tmp

FOUND=$(grep "Infected files:" "$LOG" | awk '{print $NF}')
if [ "$FOUND" -gt 0 ]; then
    mail -s "ClamAV: $FOUND infected files found" root < "$LOG"
fi
```

```bash
sudo chmod +x /etc/cron.weekly/clamav-scan
```

Excluding `/proc`, `/sys`, and `/dev` is essential — these virtual filesystems can appear to be infinite loops to a scanner, and scanning them wastes CPU while producing no useful results. For servers, also exclude database directories (e.g., `/var/lib/mysql`, `/var/lib/postgresql`) to avoid performance impacts during business hours.

If your system does not have a mail transport agent configured, replace the `mail` call with a write to a dedicated alerts file, or use a webhook to a monitoring system like Prometheus Alertmanager.

---

### Step 6: Quarantine Management

```bash
# Create quarantine directory
sudo mkdir -p /var/lib/clamav/quarantine
sudo chown clamav:clamav /var/lib/clamav/quarantine
sudo chmod 700 /var/lib/clamav/quarantine

# Delete quarantined files older than 30 days
find /var/lib/clamav/quarantine -mtime +30 -delete
```

Quarantined files retain their original filenames. Before deleting, you may want to submit unknown detections to VirusTotal for confirmation — a ClamAV detection that no other engine confirms could be a false positive. Keep the 30-day cleanup on a cron job to prevent the quarantine directory from filling disk.

---

### Step 7: Verify ClamAV Is Working

```bash
# Test with EICAR test file (harmless detection test string)
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /tmp/eicar-test.com

clamscan /tmp/eicar-test.com
# Expected: /tmp/eicar-test.com: Eicar-Signature FOUND

rm /tmp/eicar-test.com

# Check definitions are current
freshclam --version
# Shows: ClamAV X.Y.Z/NNNNN/date
```

The EICAR test string is an industry-standard harmless detection test. Every conformant antivirus engine detects it as `Eicar-Signature`. If ClamAV does not flag it, your installation has a problem — either the daemon is not loaded, signatures are not current, or the socket path is wrong.

After a production deployment, run this check monthly and pipe the result into your monitoring system. A ClamAV installation that silently stops detecting due to a signature update failure provides a false sense of security.

---

### Step 8: Scanning Email Attachments (Postfix Integration)

```bash
sudo apt install clamav-milter

# /etc/clamav/clamav-milter.conf:
MilterSocket /var/run/clamav/milter.ctl
OnInfected Reject
LogSyslog yes
LogInfected Full
```

```bash
# Add to Postfix main.cf:
# smtpd_milters = unix:/var/run/clamav/milter.ctl
# milter_default_action = accept

sudo systemctl restart postfix clamav-milter
```

`OnInfected Reject` returns a 550 SMTP error to the sending server. Use `Quarantine` instead if you want to receive infected mail but store it separately for analysis. `milter_default_action = accept` tells Postfix to deliver mail normally if the milter is unreachable — this prevents mail delivery outages if ClamAV has a problem, at the cost of allowing unscanned mail through. For high-security environments, set `milter_default_action = reject` to fail closed.

---

### Step 9: Tuning Scan Performance

For systems scanning large file trees, several options reduce ClamAV's resource footprint without sacrificing coverage:

```bash
# In clamd.conf — limit parallel scan threads
MaxThreads 4

# Reduce memory usage by disabling bytecode JIT if RAM is constrained
BytecodeMode InterpretOnly

# Exclude specific file extensions unlikely to carry threats
ExcludePath /home/.*/\.git/
ExcludePath /home/.*/node_modules/
```

The `node_modules` exclusion matters on developer workstations — a typical JavaScript project contains tens of thousands of small files that would take hours to scan and have zero malware risk. The `.git` directory exclusion similarly prevents scanning object blobs that contain historical file contents, which can trigger false positives on files that historically contained known-bad strings.

To benchmark scan time, use `time clamdscan -r --no-summary /home/user/` before and after applying exclusions. On a workstation with a typical developer home directory, the difference is often 45 minutes versus under 5 minutes — with no meaningful reduction in actual threat detection coverage.

---

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [Setting Up Fail2ban for Server Protection](/privacy-tools-guide/fail2ban-server-protection-setup-guide/)
- [How to Set Up a Honeypot for Intrusion Detection](/privacy-tools-guide/honeypot-intrusion-detection-guide/)
- [Securing Redis and MongoDB for Production](/privacy-tools-guide/securing-redis-mongodb-production-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
