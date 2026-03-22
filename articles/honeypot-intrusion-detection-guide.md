---
layout: default
title: "How to Set Up a Honeypot for Intrusion Detection"
description: "Deploy low-interaction honeypots with Cowrie SSH, OpenCanary, and HoneyDB to detect attackers on your network before they reach real systems"
date: 2026-03-22
author: theluckystrike
permalink: /honeypot-intrusion-detection-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up a Honeypot for Intrusion Detection

A honeypot is a decoy system that looks valuable but has no legitimate users. Any access to it is suspicious by definition — there's no reason for real traffic. When someone probes it, you get an early warning that an attacker is on your network or scanning your infrastructure, along with their IP, tools, and techniques.

---

## Honeypot Types

| Type | What it simulates | Complexity | Risk |
|------|------------------|-----------|------|
| Low-interaction | Fake services (SSH, HTTP) — no real shell | Low | Low |
| Medium-interaction | Realistic but limited OS environment | Medium | Medium |
| High-interaction | Real systems, carefully monitored | High | High |

For most home and small business uses, low-interaction honeypots (Cowrie, OpenCanary) are appropriate. They catch automated scanners, brute-force bots, and internal network reconnaissance.

---

## Cowrie: SSH Honeypot

Cowrie is a medium-to-low interaction SSH/Telnet honeypot that logs all attacker activity — commands run, files uploaded, credentials attempted.

### Install Cowrie

```bash
# Create dedicated user (never run honeypots as root)
sudo useradd -m -d /opt/cowrie -s /bin/bash cowrie
sudo su - cowrie

# Clone Cowrie
git clone https://github.com/cowrie/cowrie.git
cd cowrie

# Create virtual environment
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install -r requirements/requirements.txt

# Create config from template
cp cowrie.cfg.dist cowrie.cfg
```

### Configure Cowrie

```ini
# /opt/cowrie/cowrie/cowrie.cfg

[honeypot]
# Fake hostname shown to attackers
hostname = prod-db-01

# Log files
log_path = /opt/cowrie/var/log/cowrie
download_path = /opt/cowrie/var/lib/cowrie/downloads

[output_jsonlog]
enabled = true
logfile = /opt/cowrie/var/log/cowrie/cowrie.json

# Listen port (redirect real port 22 to this via iptables)
[ssh]
listen_endpoints = tcp:2222:interface=0.0.0.0
version = SSH-2.0-OpenSSH_8.4p1 Debian-5+deb11u1

[telnet]
enabled = true
listen_endpoints = tcp:2323:interface=0.0.0.0
```

### Redirect Port 22 to Cowrie

Move your real SSH to a different port, let Cowrie listen on 2222, redirect 22 to 2222:

```bash
# Move real SSH to port 2200
# /etc/ssh/sshd_config → Port 2200
sudo systemctl restart sshd

# Redirect port 22 to Cowrie
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2323

# Save iptables rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### Start Cowrie

```bash
# Start as cowrie user
sudo su - cowrie
cd /opt/cowrie
source cowrie-env/bin/activate
bin/cowrie start

# Watch live log
tail -f var/log/cowrie/cowrie.log
```

### Monitor Cowrie Attacks

```bash
# Watch live attack log
tail -f /opt/cowrie/var/log/cowrie/cowrie.json | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        e = json.loads(line)
        if e.get('eventid') in ['cowrie.login.failed', 'cowrie.command.input', 'cowrie.session.connect']:
            print(f\"{e['timestamp']} {e['eventid']} {e.get('src_ip','')} {e.get('input','')}\")
    except:
        pass
"

# Top attacking IPs
cat /opt/cowrie/var/log/cowrie/cowrie.json | \
  python3 -c "import sys,json; [print(json.loads(l).get('src_ip','')) for l in sys.stdin if l.strip()]" | \
  sort | uniq -c | sort -rn | head -20

# Most tried passwords
cat /opt/cowrie/var/log/cowrie/cowrie.json | \
  python3 -c "
import sys, json
for l in sys.stdin:
    try:
        e = json.loads(l)
        if e.get('eventid') == 'cowrie.login.failed':
            print(e.get('password',''))
    except:
        pass
" | sort | uniq -c | sort -rn | head -20
```

---

## OpenCanary: Multi-Service Honeypot

OpenCanary simulates multiple services at once: HTTP, FTP, SSH, MySQL, Redis, SMB, and more. Ideal for detecting internal network reconnaissance.

```bash
# Install OpenCanary
pip install opencanary
pip install scapy pcapy   # for packet-level modules (optional)

# Generate default config
opencanaryd --copyconfig
# Creates /etc/opencanary.conf
```

```json
// /etc/opencanary.conf (relevant sections)
{
    "device.node_id": "opencanary-001",
    "logger": {
        "class": "PyLogger",
        "kwargs": {
            "formatters": {
                "plain": {"format": "%(message)s"}
            },
            "handlers": {
                "file": {
                    "class": "logging.FileHandler",
                    "filename": "/var/log/opencanary.log"
                }
            }
        }
    },
    "ftp.enabled": true,
    "ftp.port": 21,
    "http.enabled": true,
    "http.port": 80,
    "http.banner": "Apache/2.4.41 (Ubuntu)",
    "httpproxy.enabled": false,
    "mysql.enabled": true,
    "mysql.port": 3306,
    "redis.enabled": true,
    "redis.port": 6379,
    "ssh.enabled": true,
    "ssh.port": 8022,
    "smb.enabled": false,
    "snmp.enabled": false,
    "telnet.enabled": true,
    "telnet.port": 23
}
```

```bash
# Start OpenCanary
opencanaryd --start

# Watch log in real time
tail -f /var/log/opencanary.log | python3 -m json.tool

# Run as a service
cat > /etc/systemd/system/opencanary.service << 'EOF'
[Unit]
Description=OpenCanary Honeypot
After=network.target

[Service]
ExecStart=/usr/local/bin/opencanaryd --foreground
Restart=on-failure
User=nobody

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now opencanary
```

---

## Alert on Honeypot Access

The real value of a honeypot is alerting — knowing the moment someone accesses it:

```python
#!/usr/bin/env python3
"""Alert when honeypot (OpenCanary) detects any access."""
import subprocess
import json
import smtplib
import time
from email.message import EmailMessage
from pathlib import Path

LOG_FILE = "/var/log/opencanary.log"
LAST_POS_FILE = "/tmp/.opencanary-pos"
ALERT_EMAIL = "admin@yourdomain.com"

def send_alert(event: dict):
    msg = EmailMessage()
    msg['Subject'] = f'HONEYPOT ALERT: {event.get("dst_service", "unknown")} access from {event.get("src_host", "unknown")}'
    msg['From'] = 'honeypot@yourdomain.com'
    msg['To'] = ALERT_EMAIL
    msg.set_content(f"""
Honeypot access detected!

Time: {event.get('local_time', 'unknown')}
Source IP: {event.get('src_host', 'unknown')}
Service: {event.get('dst_service', 'unknown')}
Port: {event.get('dst_port', 'unknown')}
Node: {event.get('node_id', 'unknown')}

Full event:
{json.dumps(event, indent=2)}
""")
    with smtplib.SMTP('localhost') as s:
        s.send_message(msg)
    print(f"Alert sent for {event.get('src_host')} → {event.get('dst_service')}")

def get_last_pos():
    p = Path(LAST_POS_FILE)
    return int(p.read_text()) if p.exists() else 0

def set_last_pos(pos):
    Path(LAST_POS_FILE).write_text(str(pos))

def monitor():
    while True:
        pos = get_last_pos()
        with open(LOG_FILE) as f:
            f.seek(pos)
            for line in f:
                line = line.strip()
                if not line:
                    continue
                try:
                    event = json.loads(line)
                    # Any access is suspicious — alert on all events
                    send_alert(event)
                except json.JSONDecodeError:
                    pass
            set_last_pos(f.tell())
        time.sleep(30)

if __name__ == "__main__":
    monitor()
```

---

## Canary Tokens: File-Based Honeypots

Canary tokens are files (documents, PDFs, images) that phone home when opened. Drop them in sensitive directories — if someone opens the "Passwords.xlsx" canary token, you get notified.

```bash
# Generate a canary token at canarytokens.org
# Choose: Word Document, PDF, or Windows Folder

# Download the token file
wget https://canarytokens.org/download?token=abc123... -O "Passwords-2026.docx"

# Place in directories you want to monitor:
cp "Passwords-2026.docx" ~/Documents/
cp "Passwords-2026.docx" /var/www/html/admin/
cp "Passwords-2026.docx" /home/backup/credentials/

# When opened from anywhere, you receive an email alert with:
# - Timestamp
# - IP address of the opener
# - Browser/OS details (for web tokens)
```

---

## Network Canary: Detect Internal Reconnaissance

Place a canary service on an IP that should never be accessed:

```bash
# Use a spare IP on your LAN that no legitimate device uses
# Set up OpenCanary on this IP
# Any port scan or connection attempt = intruder alert

# On your router, assign the canary IP as static
# In Pi-hole: block the canary's own DNS queries (avoid false positives)

# Monitor with nftables logging:
sudo nft add rule inet filter input ip daddr 192.168.1.200 \
  log prefix "CANARY_HIT: " counter drop
sudo journalctl -f | grep CANARY_HIT
```

---

## Related Reading

- [Setting Up Fail2ban for Server Protection](/privacy-tools-guide/fail2ban-server-protection-setup-guide/)
- [Network Segmentation for IoT Devices](/privacy-tools-guide/iot-network-segmentation-vlan-guide/)
- [How to Monitor Dark Web for Data Breaches](/privacy-tools-guide/dark-web-breach-monitoring-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)


## Frequently Asked Questions


**How long does it take to set up a honeypot for intrusion detection?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.


**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.


**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.


**Will this work with my existing CI/CD pipeline?**

The core concepts apply across most CI/CD platforms, though specific syntax and configuration differ. You may need to adapt file paths, environment variable names, and trigger conditions to match your pipeline tool. The underlying workflow logic stays the same.


**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.


{% endraw %}
