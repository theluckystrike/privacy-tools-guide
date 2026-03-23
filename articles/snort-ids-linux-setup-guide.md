---
layout: default
title: "How to Set Up Snort IDS on Linux"
description: "Step-by-step guide to installing and configuring Snort intrusion detection on Linux with custom rules, logging, and real-time alerts"
date: 2026-03-22
author: theluckystrike
permalink: /snort-ids-linux-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# How to Set Up Snort IDS on Linux

Snort is the most widely deployed open-source intrusion detection system. It inspects network packets in real time against a library of rules, logs suspicious traffic, and fires alerts when threats match. This guide walks through a full production-ready setup on Ubuntu 22.04/24.04 using Snort 3.

## Prerequisites

- Ubuntu 22.04 or 24.04 (root or sudo access)
- A dedicated network interface for monitoring (can be the same interface you SSH through initially)
- At least 2 GB RAM; 4 GB recommended for busy networks
- Stable internet connection for downloading rules

---

## 1. Install Dependencies

Snort 3 requires a handful of libraries not in the default Ubuntu repos.

```bash
sudo apt update && sudo apt install -y \
  build-essential libpcap-dev libpcre3-dev libdnet-dev \
  zlib1g-dev liblzma-dev openssl libssl-dev libhwloc-dev \
  pkg-config cmake ninja-build git curl wget flex bison \
  libluajit-5.1-dev luajit libsafec-dev uuid-dev

# LibDAQ — Snort's packet acquisition library
cd /tmp
git clone https://github.com/snort3/libdaq.git
cd libdaq && ./bootstrap && ./configure --prefix=/usr/local && make -j$(nproc)
sudo make install && sudo ldconfig
```

---

## 2. Install Snort 3

```bash
cd /tmp
wget https://github.com/snort3/snort3/archive/refs/tags/3.3.7.0.tar.gz
tar xzf 3.3.7.0.tar.gz && cd snort3-3.3.7.0

./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc
cd build && make -j$(nproc)
sudo make install && sudo ldconfig

# Verify
snort --version
```

You should see output beginning with ` ,,_ -*> Snort++ <*-`.

---

## 3. Configure Network Interface

Put your monitoring interface into promiscuous mode and disable offloading features that can corrupt packet inspection:

```bash
IFACE=eth0   # replace with your interface name

sudo ip link set $IFACE promisc on

# Disable offloading so Snort sees raw packets, not reassembled ones
sudo ethtool -K $IFACE gro off lro off tso off gso off

# Make changes survive reboot — add to /etc/rc.local or a systemd unit
```

To persist promiscuous mode across reboots, create a systemd service:

```ini
# /etc/systemd/system/snort-promisc.service
[Unit]
Description=Set interface promiscuous and disable offloading for Snort
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ip link set eth0 promisc on
ExecStart=/usr/sbin/ethtool -K eth0 gro off lro off tso off gso off

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now snort-promisc
```

---

## 4. Directory Structure

```bash
sudo mkdir -p /etc/snort/rules /etc/snort/so_rules /var/log/snort
sudo chmod 700 /var/log/snort
```

---

## 5. Download and Install Rules

Snort ships with community rules (free) and registered/subscriber rules (free registration or paid). Start with community:

```bash
# Community rules
cd /tmp
wget https://www.snort.org/downloads/community/snort3-community-rules.tar.gz
tar xzf snort3-community-rules.tar.gz
sudo cp snort3-community-rules/snort3-community.rules /etc/snort/rules/

# If you have a registered Oinkcode (free at snort.org/users/sign_up):
OINKCODE=your_code_here
wget "https://www.snort.org/rules/snortrules-snapshot-3160.tar.gz?oinkcode=${OINKCODE}" \
  -O snortrules.tar.gz
tar xzf snortrules.tar.gz
sudo cp rules/*.rules /etc/snort/rules/
```

---

## 6. Main Snort Configuration

The primary config file lives at `/etc/snort/snort.lua`. Create a working baseline:

```lua
-- /etc/snort/snort.lua

-- Home network definition
HOME_NET = '192.168.0.0/16,10.0.0.0/8,172.16.0.0/12'
EXTERNAL_NET = '!$HOME_NET'

-- Rule paths
RULE_PATH      = '/etc/snort/rules'
SO_RULE_PATH   = '/etc/snort/so_rules'
BUILTIN_RULE_PATH = '/usr/local/lib/snort'

-- IPS engine — alert mode only (change to 'drop' with inline/NFQ)
ips =
{
    enable_builtin_rules = true,
    rules = [[
        include $RULE_PATH/snort3-community.rules
    ]],
    mode = 'alert',
}

-- Logging
alert_fast =
{
    file = true,
    packet = false,
}

-- Output to unified2 for integration with SIEMs
unified2 = {}

-- Detection engine settings
detection =
{
    hyperscan_literals = true,
    pcre_to_regex = true,
}

-- Packet decoder
stream = {}
stream_tcp = { policy = 'windows', ports = { client = '1024:', server = '0:1023' } }
stream_udp = {}

-- Application layer preprocessors
http_inspect = {}
ftp_telnet = {}
smtp = {}
ssh = {}
dns = {}

-- Logging verbosity (0–5)
-- trace = { modules = { detection = { rule_engine = 1 } } }
```

---

## 7. Write a Local Custom Rule

Custom rules let you detect traffic specific to your environment. Rules live in a separate file to survive community rule updates:

```bash
sudo tee /etc/snort/rules/local.rules > /dev/null <<'EOF'
# Alert on any Telnet connection attempt from external
alert tcp $EXTERNAL_NET any -> $HOME_NET 23 \
  (msg:"TELNET connection attempt from external"; \
   flow:to_server,established; \
   sid:9000001; rev:1;)

# Alert on ICMP flood (>5 pings per second heuristic)
alert icmp $EXTERNAL_NET any -> $HOME_NET any \
  (msg:"ICMP flood detected"; \
   threshold:type threshold, track by_src, count 5, seconds 1; \
   sid:9000002; rev:1;)

# Alert on SSH brute-force (10 attempts in 60 seconds)
alert tcp $EXTERNAL_NET any -> $HOME_NET 22 \
  (msg:"SSH brute force attempt"; \
   flow:to_server; flags:S; \
   threshold:type threshold, track by_src, count 10, seconds 60; \
   sid:9000003; rev:1;)
EOF
```

Add the local rules to `snort.lua` by appending inside the `ips.rules` block:

```lua
include $RULE_PATH/local.rules
```

---

## 8. Test Configuration

```bash
sudo snort -c /etc/snort/snort.lua --plugin-path /usr/local/lib/snort \
  --warn-all -T

# Expected: "Snort successfully validated the configuration"
```

---

## 9. Run Snort

**Test against a PCAP file first** (does not touch live traffic):

```bash
sudo snort -c /etc/snort/snort.lua -r /path/to/test.pcap -l /var/log/snort
```

**Live interface monitoring:**

```bash
sudo snort -c /etc/snort/snort.lua -i eth0 -l /var/log/snort -D \
  --pid-path /var/run/snort.pid
```

The `-D` flag daemonizes Snort. Logs appear in `/var/log/snort/alert_fast.txt`.

---

## 10. Systemd Service

```ini
# /etc/systemd/system/snort.service
[Unit]
Description=Snort3 Intrusion Detection System
After=network.target snort-promisc.service
Requires=snort-promisc.service

[Service]
Type=simple
ExecStart=/usr/local/bin/snort -c /etc/snort/snort.lua \
  -i eth0 -l /var/log/snort --plugin-path /usr/local/lib/snort
Restart=on-failure
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now snort
sudo systemctl status snort
```

---

## 11. Reading Alerts

```bash
# Tail live alerts
sudo tail -f /var/log/snort/alert_fast.txt

# Count by rule message
sudo awk -F'\\[\\*\\*\\]' '{print $2}' /var/log/snort/alert_fast.txt \
  | sort | uniq -c | sort -rn | head -20
```

A typical fast alert line looks like:

```
03/22-14:22:01.001234 [**] [1:9000003:1] "SSH brute force attempt" [**] \
[Priority: 2] {TCP} 203.0.113.5:54321 -> 192.168.1.10:22
```

---

## 12. Automatic Rule Updates with PulledPork

```bash
# Install PulledPork 3 (Python-based)
pip3 install pulledpork3

# Configure: edit /etc/pulledpork/pulledpork.conf
# Set oinkcode, rule_path, snort_path, and pid_path

# Run update (add to daily cron)
sudo pulledpork -c /etc/pulledpork/pulledpork.conf
sudo systemctl restart snort
```

---

## Tuning Tips

- **Suppress noisy rules**: Use `suppress` blocks in `snort.lua` to silence rules that generate false positives in your environment.
- **Rate limiting**: The `threshold` keyword prevents alert floods from a single source.
- **VLAN tagging**: Add `--daq-var snaplen=65535` if monitoring a trunked interface.
- **Memory limits**: Set `--max-packet-threads` to match your CPU cores without exhausting RAM.
- **Integration**: Forward `/var/log/snort/unified2` output to Splunk, ELK, or Wazuh using `barnyard2` or the unified2 reader.

---

## Related Articles

- [Suricata Home Network IDS Setup Guide](/suricata-home-network-ids-setup/)
- [Nextcloud Setup Guide Raspberry Pi 2026](/nextcloud-setup-guide-raspberry-pi-2026/)
- [How to Set Up Mullvad VPN on Linux](/mullvad-vpn-linux-setup-guide/)
- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)
- [LUKS Full Disk Encryption on Linux](/luks-full-disk-encryption-linux-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
