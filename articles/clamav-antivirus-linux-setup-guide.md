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

---

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

---

## Update Virus Definitions

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

**Configure freshclam:**

```bash
sudo nano /etc/clamav/freshclam.conf

# Key settings:
Checks 24                              # Check 24 times per day
LogFile /var/log/clamav/freshclam.log
NotifyClamd /etc/clamav/clamd.conf
```

---

## Basic Scanning

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

---

## Configure the ClamAV Daemon (clamd)

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

---

## On-Access Scanning (Real-Time)

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

---

## Scheduled Scans

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

---

## Quarantine Management

```bash
# Create quarantine directory
sudo mkdir -p /var/lib/clamav/quarantine
sudo chown clamav:clamav /var/lib/clamav/quarantine
sudo chmod 700 /var/lib/clamav/quarantine

# Delete quarantined files older than 30 days
find /var/lib/clamav/quarantine -mtime +30 -delete
```

---

## Verify ClamAV Is Working

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

---

## Scanning Email Attachments (Postfix Integration)

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

---

## Related Reading

- [Setting Up Fail2ban for Server Protection](/privacy-tools-guide/fail2ban-server-protection-setup-guide/)
- [How to Set Up a Honeypot for Intrusion Detection](/privacy-tools-guide/honeypot-intrusion-detection-guide/)
- [Securing Redis and MongoDB for Production](/privacy-tools-guide/securing-redis-mongodb-production-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
