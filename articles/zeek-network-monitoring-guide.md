---
layout: default
title: "How to Use Zeek for Network Monitoring"
description: "Deploy Zeek on Linux for deep network traffic analysis, custom detection scripts, protocol logging, and integration with your SIEM or security pipeline"
date: 2026-03-22
author: theluckystrike
permalink: /zeek-network-monitoring-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# How to Use Zeek for Network Monitoring

Zeek (formerly Bro) does not match patterns against packets — it reconstructs full sessions and writes structured logs for every protocol it understands. You get a detailed record of every DNS query, HTTP request, TLS handshake, and connection on your network, queryable without replaying PCAPs. This guide covers installation, log analysis, and writing custom detection scripts.

## How Zeek Differs from Snort/Suricata

Snort and Suricata match signatures against raw packet bytes. Zeek parses protocols into structured events and exposes them to a scripting engine. The result: Zeek logs are SQL-queryable, schema-consistent, and much richer than IDS alerts — but Zeek doesn't block traffic by default.

Zeek is commonly paired with a SIEM (Splunk, Elastic, Wazuh) as the data source for threat hunting, rather than as a real-time blocker.

---

## 1. Install Zeek

```bash
# Ubuntu 24.04 — use the official repository
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' \
  | sudo tee /etc/apt/sources.list.d/security:zeek.list

curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update && sudo apt install -y zeek

# Add Zeek to PATH
echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

zeek --version
```

---

## 2. Configure Interface and Logging

```bash
# Edit the main node config
sudo tee /opt/zeek/etc/node.cfg > /dev/null <<'EOF'
[zeek]
type=standalone
host=localhost
interface=eth0   # replace with your capture interface
EOF

# Set log rotation and format
sudo tee -a /opt/zeek/etc/zeekctl.cfg > /dev/null <<'EOF'
LogRotationInterval = 3600
LogDir = /var/log/zeek
EOF

# Create log dir
sudo mkdir -p /var/log/zeek
sudo chown zeek: /var/log/zeek

# Disable interface offloading
sudo ethtool -K eth0 rx off tx off sg off tso off gso off gro off lro off
```

---

## 3. Deploy with ZeekControl

```bash
# Install (first-time setup)
sudo zeekctl install

# Start
sudo zeekctl start

# Check status
sudo zeekctl status

# Stop
sudo zeekctl stop

# Zeek restarts automatically on crash; check logs:
sudo zeekctl diag
```

---

## 4. Understand the Log Files

Zeek writes tab-separated logs to `/var/log/zeek/current/`:

```bash
ls /var/log/zeek/current/
# conn.log      — every TCP/UDP/ICMP connection
# dns.log       — every DNS query and response
# http.log      — every HTTP request
# ssl.log       — every TLS handshake
# files.log     — every transferred file
# weird.log     — protocol anomalies
# notice.log    — policy-triggered notices
# dpd.log       — dynamic protocol detection results
```

Read logs with `zeek-cut` (Zeek's column selector):

```bash
# Top 10 source IPs by connection count
zeek-cut id.orig_h < /var/log/zeek/current/conn.log \
  | sort | uniq -c | sort -rn | head -10

# All DNS queries for a specific domain
zeek-cut ts id.orig_h query answers < /var/log/zeek/current/dns.log \
  | grep "example\.com"

# TLS connections with expired or self-signed certificates
zeek-cut ts id.orig_h id.resp_h server_name validation_status \
  < /var/log/zeek/current/ssl.log \
  | grep -v "ok"

# HTTP requests with non-200 status and large response bodies (potential data exfil)
zeek-cut ts id.orig_h host uri status_code resp_body_len \
  < /var/log/zeek/current/http.log \
  | awk -F'\t' '$6 > 1000000'
```

---

## 5. Write a Custom Detection Script

Zeek scripts are event-driven. This script detects SSH logins from unexpected countries by watching for repeated connection attempts and fires a notice:

```zeek
# /opt/zeek/share/zeek/site/detect-ssh-scan.zeek

module SSHScan;

export {
    redef enum Notice::Type += {
        SSH_Scanner,
    };
    const scan_threshold = 5 &redef;
    const scan_interval  = 60 sec &redef;
}

global ssh_attempts: table[addr] of count &default=0 &create_expire=1 min;

event ssh_auth_attempted(c: connection, authenticated: bool)
{
    if ( !authenticated )
    {
        local src = c$id$orig_h;
        ++ssh_attempts[src];

        if ( ssh_attempts[src] == scan_threshold )
        {
            NOTICE([
                $note=SSH_Scanner,
                $conn=c,
                $msg=fmt("%s made %d failed SSH attempts in %s",
                         src, ssh_attempts[src], scan_interval),
                $identifier=cat(src),
            ]);
        }
    }
}
```

Load it in `local.zeek`:

```bash
echo '@load site/detect-ssh-scan' | sudo tee -a /opt/zeek/share/zeek/site/local.zeek
sudo zeekctl deploy
```

---

## 6. Detect DNS Tunneling

DNS tunneling encodes data in DNS queries to exfiltrate traffic or bypass firewalls. Telltale signs: very long query names, high query rate, unusual record types.

```zeek
# /opt/zeek/share/zeek/site/detect-dns-tunnel.zeek

module DNSTunnel;

export {
    redef enum Notice::Type += {
        Possible_DNS_Tunnel,
    };
}

event dns_request(c: connection, msg: dns_msg, qtype: count, qclass: count)
{
    if ( |c$dns$query| > 50 )
    {
        NOTICE([
            $note=Possible_DNS_Tunnel,
            $conn=c,
            $msg=fmt("Long DNS query (%d chars): %s from %s",
                     |c$dns$query|, c$dns$query, c$id$orig_h),
        ]);
    }
}
```

---

## 7. Integrate with Elasticsearch (ELK)

```bash
# Install Filebeat
sudo apt install -y filebeat

# Use Zeek module
sudo filebeat modules enable zeek

# Edit /etc/filebeat/modules.d/zeek.yml
# Set path for each log type:
# - module: zeek
#   connection:
#     enabled: true
#     var.paths: ["/var/log/zeek/current/conn.log"]

sudo filebeat setup --index-management -E output.logstash.enabled=false \
  -E 'output.elasticsearch.hosts=["http://localhost:9200"]'

sudo systemctl enable --now filebeat
```

Kibana provides a pre-built Zeek dashboard once data flows in.

---

## 8. Query Logs with `zq` (Fast JSON/TSV Tool)

`zq` (from the Zed project) queries Zeek TSV logs like a database:

```bash
# Install
sudo snap install zq || pip3 install zq

# Find all HTTP requests to a specific host
zq -f text 'host=="malicious.example.com" | count()' /var/log/zeek/current/http.log

# Top talkers by bytes sent
zq -f text 'sum(orig_bytes) by id.orig_h | sort -r sum | head 10' \
  /var/log/zeek/current/conn.log

# DNS queries for newly-registered domains (requires threat intel enrichment)
zq -f text 'qtype_name=="A" | count() by query | sort -r count | head 20' \
  /var/log/zeek/current/dns.log
```

---

## Zeek Package Manager

```bash
# Install ZeekPkg
pip3 install zkg
zkg autoconfig

# Install threat intelligence framework
zkg install zeek/zeek-intel-threatbus

# Install JA3 TLS fingerprinting (identify malware by TLS client fingerprint)
zkg install salesforce/ja3

sudo zeekctl deploy
```

---

## Performance Tuning

| Setting | File | Recommendation |
|---|---|---|
| `workers` | `node.cfg` | 1 per 2 CPU cores; increase for >1 Gbps |
| `LogFlushInterval` | `zeekctl.cfg` | 60s default; reduce to 10s for real-time SIEM |
| Disable unused analyzers | `local.zeek` | `redef dpd_buffer_size = 0;` on known-quiet networks |
| AF_PACKET | `node.cfg` | `lb_method=af_packet` for multi-worker packet balancing |

---

## Related Reading

- [How to Set Up Snort IDS on Linux](/snort-ids-linux-setup-guide/)
- [How to Set Up Wazuh SIEM for Small Teams](/wazuh-siem-small-teams-setup-guide/)
- [How to Detect ARP Spoofing Attacks](/detect-arp-spoofing-attacks-guide/)
- [How to Monitor the Dark Web for Data Breaches](/dark-web-breach-monitoring-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [Enterprise AI Coding Tool Network Security Requirements](https://bestremotetools.com/enterprise-ai-coding-tool-network-security-requirements-and-/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
