---
layout: default
title: "Suricata Home Network IDS Setup Guide"
description: "How to set up Suricata as a network intrusion detection system on a Linux home server or router to monitor for malware, data exfiltration, and suspicious"
date: 2026-03-21
author: theluckystrike
permalink: /suricata-home-network-ids-setup/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Suricata is an open-source network threat detection engine that can analyze all traffic passing through your home network in real time. It identifies known malware signatures, suspicious connection patterns, data exfiltration attempts, and policy violations using rule sets maintained by the community and commercial security teams.

This guide sets up Suricata in IDS (Intrusion Detection System) mode on a Linux host that sees your network traffic — either a router running OpenWrt, a Pi-hole server, or any always-on Linux machine positioned to see your LAN traffic.

## Architecture Options

**Inline mode (IPS)**: Suricata sits in the traffic path and can drop packets. Requires the machine to be a router or bridge device.

**Passive/sniffing mode (IDS)**: Suricata listens on a network interface and analyzes a copy of traffic. Cannot block but generates alerts. Suitable for any machine on your network.

This guide covers passive mode — the safer starting point. You see what's happening without risking network disruption from misconfigured drop rules.

## Installation

### Debian/Ubuntu

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update
sudo apt install suricata

# Verify installation
suricata --version
```

### From source (latest version)

```bash
sudo apt install libpcre3-dev libpcre3 libnet1-dev libyaml-dev \
  libcap-ng-dev libmagic-dev libjansson-dev libnspr4-dev \
  libnss3-dev liblz4-dev rustc cargo

# Download and build
wget https://www.openinfosecfoundation.org/download/suricata-7.0.x.tar.gz
tar -xzf suricata-7.0.x.tar.gz && cd suricata-7.0.x
./configure --enable-nfqueue --prefix=/usr --sysconfdir=/etc --localstatedir=/var
make && sudo make install-conf
```

## Step 1: Configure Suricata

The main configuration file is `/etc/suricata/suricata.yaml`. Key sections to configure:

```bash
sudo nano /etc/suricata/suricata.yaml
```

### Set your home network range

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
    EXTERNAL_NET: "!$HOME_NET"
    # Add other segments specific to your network
```

### Configure the capture interface

```yaml
af-packet:
  - interface: eth0   # Replace with your LAN-facing interface
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
    use-mmap: yes
    tpacket-v3: yes
```

Check your interface name:

```bash
ip link show
# Look for your LAN interface (commonly eth0, enp3s0, br-lan on routers)
```

### Configure logging

```yaml
outputs:
  - fast:
      enabled: yes
      filename: /var/log/suricata/fast.log
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert:
            metadata: yes
            tagged-packets: yes
        - dns:
            query: yes
            answer: yes
        - http:
            extended: yes
        - tls:
            extended: yes
        - files:
            force-magic: no
        - netflow:
            enabled: no
```

## Step 2: Install Rule Sets

Suricata needs rule files to detect threats. Use `suricata-update` to manage rules:

```bash
# Update the default rule sources
sudo suricata-update update-sources

# List available rule sources
sudo suricata-update list-sources

# Enable free high-quality rule sets
sudo suricata-update enable-source et/open          # Emerging Threats Open
sudo suricata-update enable-source oisf/trafficid   # Traffic identification

# Update rules
sudo suricata-update

# Verify rules loaded
sudo suricata-update list-enabled-sources
ls /var/lib/suricata/rules/
```

The Emerging Threats Open ruleset contains over 35,000 rules covering known malware, C2 communication, exploit attempts, and suspicious behavior patterns.

## Step 3: Test the Configuration

```bash
# Test configuration file for syntax errors
sudo suricata -T -c /etc/suricata/suricata.yaml

# If output shows: "Configuration provided was successfully loaded."
# the configuration is valid.
```

## Step 4: Start Suricata

```bash
# Start in IDS mode on your interface
sudo systemctl enable suricata
sudo systemctl start suricata

# Check it's running
sudo systemctl status suricata

# Watch alerts in real time
sudo tail -f /var/log/suricata/fast.log
```

You should see something like:
```
03/21/2026-14:23:01.123456  [**] [1:2001219:20] ET SCAN Potential SSH Scan [**] [Classification: Attempted Information Leak] [Priority: 2] {TCP} 192.168.1.50:52341 -> 93.184.216.34:22
```

## Step 5: Test with a Known-Bad Signature

Trigger a test rule to verify detection is working:

```bash
# Suricata includes a test rule (sid 2100498) that triggers on a known test string
# Send it as an HTTP request to trigger the rule
curl http://testmyids.com/

# Check for the alert
sudo grep "2100498" /var/log/suricata/fast.log
```

## Step 6: Analyze Alerts with jq

The EVE JSON log is machine-readable and can be analyzed with `jq`:

```bash
# View all alerts from the last hour
sudo jq 'select(.event_type=="alert")' /var/log/suricata/eve.json | \
  jq '{timestamp, src_ip, dest_ip, proto, alert: .alert.signature}'

# Count alerts by signature
sudo cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert") | .alert.signature' | \
  sort | uniq -c | sort -rn | head -20

# Find all DNS queries made by a specific IP
sudo jq 'select(.event_type=="dns" and .src_ip=="192.168.1.50")' \
  /var/log/suricata/eve.json | jq '.dns.rrname'

# Find all TLS connections to unusual ports
sudo jq 'select(.event_type=="tls" and .dest_port!=443)' \
  /var/log/suricata/eve.json | jq '{timestamp, src_ip, dest_ip, dest_port, tls}'
```

## Step 7: Suppress False Positives

New Suricata installations generate many false positives. Suppress rules that are noisy and irrelevant to your environment:

Create `/etc/suricata/threshold.conf`:

```
# Suppress a specific rule (by SID) for all sources
suppress gen_id 1, sig_id 2013028

# Suppress a rule for a specific source IP (e.g., your media server)
suppress gen_id 1, sig_id 2012887, track by_src, ip 192.168.1.200

# Rate-limit alerts from a specific rule
threshold gen_id 1, sig_id 2009582, type threshold, track by_src, count 5, seconds 60
```

Reference this file in `suricata.yaml`:
```yaml
threshold-file: /etc/suricata/threshold.conf
```

## Step 8: Automate Rule Updates

```bash
# Set up daily rule updates via cron
echo "0 2 * * * root /usr/bin/suricata-update && systemctl reload suricata" \
  | sudo tee /etc/cron.d/suricata-update
```

## Viewing Logs with Kibana or Grafana (Optional)

For a graphical dashboard, the ELK Stack (Elasticsearch + Logstash + Kibana) or Grafana + Loki can ingest Suricata's EVE JSON logs:

```bash
# Quick setup with Filebeat to ship logs to Elasticsearch
sudo apt install filebeat
sudo filebeat modules enable suricata
sudo filebeat setup
sudo systemctl start filebeat
```

Kibana provides pre-built dashboards for Suricata that show alert trends, top sources, protocol breakdowns, and DNS analysis.

## Related Reading

- [Wireshark Basics for Privacy Network Analysis](/wireshark-privacy-network-analysis-guide/)
- [How to Set Up a Privacy-Focused Home Server](/how-to-set-up-secure-home-server-for-self-hosting-privacy-tools/)
- [Pi-hole DNS Filtering Guide 2026](/home-network-privacy-pihole-dns-filtering-guide-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
