---
layout: default
title: "How to Use OSSEC for Host Intrusion Detection"
description: "Install OSSEC as a host-based IDS, configure rules for file integrity monitoring, log analysis, and rootcheck, and set up email alerts for security events"
date: 2026-03-22
author: theluckystrike
permalink: /ossec-host-intrusion-detection-setup/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Use OSSEC for Host Intrusion Detection

OSSEC is a host-based intrusion detection system (HIDS) that monitors your server in real time. It watches log files for suspicious patterns, checks file integrity against known-good baselines, runs rootkit detection scans, and sends alerts when something changes. Unlike network IDS (which monitors traffic), OSSEC runs on the host and sees inside the system.

## Key Takeaways

- **Topics covered**: what ossec monitors, installation, configuration overview
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## What OSSEC Monitors

- **File Integrity Monitoring (FIM):** Alerts when a monitored file is added, modified, or deleted
- **Log Analysis:** Scans system logs for attack patterns (SSH brute force, su attempts, Apache errors)
- **Rootcheck:** Detects rootkits, hidden files, and suspicious kernel modules
- **Policy monitoring:** Checks for insecure configurations

## Installation

```bash
# Install dependencies
sudo apt install build-essential libpcre2-dev libz-dev libssl-dev

# Download and verify
wget https://github.com/ossec/ossec-hids/archive/refs/tags/3.7.0.tar.gz
sha256sum 3.7.0.tar.gz
# Verify against published hash at ossec.net

tar xzf 3.7.0.tar.gz
cd ossec-hids-3.7.0

# Interactive installer
sudo ./install.sh
```

During installation:
- Choose **local** for a standalone system (or **server** if you have multiple agents)
- Accept defaults for directories: `/var/ossec`
- When asked about email, configure now or leave blank and edit config later
- Enable active response for common attacks (optional, review before enabling)

```bash
# Start OSSEC
sudo /var/ossec/bin/ossec-control start

# Verify it is running
sudo /var/ossec/bin/ossec-control status
# ossec-monitord is running...
# ossec-logcollector is running...
# ossec-syscheckd is running...
# ossec-analysisd is running...
```

## Configuration Overview

All configuration is in `/var/ossec/etc/ossec.conf`:

```xml
<!-- /var/ossec/etc/ossec.conf — key sections -->

<ossec_config>

  <!-- Global settings: email alerts -->
  <global>
    <email_notification>yes</email_notification>
    <email_to>admin@yourdomain.com</email_to>
    <smtp_server>127.0.0.1</smtp_server>
    <email_from>ossec@yourserver.example.com</email_from>
    <!-- Alert level threshold for email (0-15, default 7) -->
    <email_alert_level>7</email_alert_level>
  </global>

  <!-- File Integrity Monitoring -->
  <syscheck>
    <frequency>7200</frequency>  <!-- Check every 2 hours -->
    <scan_on_start>yes</scan_on_start>
    <alert_new_files>yes</alert_new_files>

    <!-- Directories to monitor -->
    <directories check_all="yes">/etc</directories>
    <directories check_all="yes">/usr/bin,/usr/sbin</directories>
    <directories check_all="yes">/bin,/sbin</directories>
    <directories check_all="yes" report_changes="yes">/var/www/html</directories>

    <!-- Ignore volatile files to reduce false positives -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/mail/statistics</ignore>
    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/adjtime</ignore>
  </syscheck>

  <!-- Rootcheck configuration -->
  <rootcheck>
    <rootkit_files>/var/ossec/etc/shared/rootkit_files.txt</rootkit_files>
    <rootkit_trojans>/var/ossec/etc/shared/rootkit_trojans.txt</rootkit_trojans>
    <system_audit>/var/ossec/etc/shared/system_audit_rcl.txt</system_audit>
    <frequency>43200</frequency>  <!-- Every 12 hours -->
  </rootcheck>

  <!-- Log files to monitor -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
  </localfile>

  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
  </localfile>

  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/apache2/access.log</location>
  </localfile>

  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/nginx/access.log</location>
  </localfile>

</ossec_config>
```

## Adding Custom Rules

OSSEC rules are XML. Custom rules go in `/var/ossec/rules/local_rules.xml`.

```xml
<!-- /var/ossec/rules/local_rules.xml -->
<group name="local,custom,">

  <!-- Alert on any sudo usage -->
  <rule id="100001" level="6">
    <if_sid>5401</if_sid>
    <description>Sudo command executed</description>
    <group>sudo,</group>
  </rule>

  <!-- Alert on new SSH key files added to authorized_keys -->
  <rule id="100002" level="12">
    <if_sid>554</if_sid>
    <match>authorized_keys</match>
    <description>SSH authorized_keys file modified</description>
    <group>authentication,</group>
  </rule>

  <!-- Alert on cron job changes -->
  <rule id="100003" level="8">
    <if_sid>554</if_sid>
    <match>/etc/cron</match>
    <description>Cron configuration modified</description>
  </rule>

  <!-- Ignore noisy rule (lower its level) -->
  <rule id="100010" level="0">
    <if_sid>5503</if_sid>
    <description>Ignore specific noisy rule</description>
  </rule>

</group>
```

```bash
# Validate rules and restart
sudo /var/ossec/bin/ossec-control restart

# Test a rule against sample log data
echo "Mar 22 10:30:00 server sshd[1234]: Invalid user admin from 185.234.219.45" \
  | sudo /var/ossec/bin/ossec-logtest
```

## Checking Alerts

```bash
# Recent alerts
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Only high-severity alerts (level 7+)
sudo grep "level=\"[7-9]\|level=\"1[0-5]" /var/ossec/logs/alerts/alerts.xml | head -40

# File integrity changes
sudo cat /var/ossec/logs/alerts/alerts.log | grep "Integrity checksum changed"

# Parse with Python
python3 << 'EOF'
import xml.etree.ElementTree as ET
import sys

with open('/var/ossec/logs/alerts/alerts.xml') as f:
    # OSSEC XML is not well-formed (multiple roots), wrap it
    content = '<root>' + f.read() + '</root>'

try:
    root = ET.fromstring(content)
    for alert in root.findall('alert'):
        level = alert.findtext('rule/level', '0')
        if int(level) >= 8:
            desc = alert.findtext('rule/description', '')
            time = alert.findtext('date', '')
            print(f"[{level}] {time}: {desc}")
except ET.ParseError as e:
    print(f"Parse error: {e}")
EOF
```

## Active Response

OSSEC can automatically block IPs that trigger certain rules (like SSH brute force):

```xml
<!-- In ossec.conf -->
<active-response>
  <command>host-deny</command>
  <location>local</location>
  <rules_id>5720</rules_id>   <!-- SSH brute force rule ID -->
  <level>8</level>
  <timeout>600</timeout>       <!-- Block for 10 minutes -->
</active-response>
```

```bash
# Verify active response scripts exist
ls /var/ossec/active-response/bin/
# host-deny.sh  firewall-drop.sh  route-null.sh  etc.

# Test active response
sudo /var/ossec/active-response/bin/host-deny.sh add "" 185.234.219.45 0
```

Review active response carefully before enabling in production — false positives can lock out legitimate users.

## Scheduled Syscheck Reports

```bash
# Generate a file integrity report
sudo /var/ossec/bin/syscheck_control -u 0  # check agent 0 (local)
sudo /var/ossec/bin/syscheck_control -l    # list monitored files

# Export database of baseline hashes
sudo /var/ossec/bin/manage_agents -l

# Run rootcheck manually
sudo /var/ossec/bin/rootcheck_control -u 0
```

## Log Rotation

OSSEC logs grow fast:

```bash
# /etc/logrotate.d/ossec
/var/ossec/logs/alerts/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        /var/ossec/bin/ossec-control restart > /dev/null 2>&1 || true
    endscript
}
```

## Related Reading

- [Lynis Linux Security Audit Guide](/privacy-tools-guide/lynis-linux-security-audit-guide/)
- [How to Use Tripwire for File Integrity Monitoring](/privacy-tools-guide/tripwire-file-integrity-monitoring-guide/)
- [How to Use chkrootkit and rkhunter](/privacy-tools-guide/chkrootkit-rkhunter-rootkit-detection-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
