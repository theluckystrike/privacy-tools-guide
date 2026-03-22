---
layout: default
title: "How to Set Up CrowdSec for Server Security"
description: "Install and configure CrowdSec on Linux to block brute-force, bad bots, and CVE scanners using collaborative threat intelligence and real-time bouncers"
date: 2026-03-22
author: theluckystrike
permalink: /crowdsec-server-security-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}
# How to Set Up CrowdSec for Server Security

CrowdSec is a modern, open-source intrusion prevention system that learns from attacks across thousands of deployments. When one node gets attacked, the offending IP is shared with the community and blocked everywhere. This guide covers installing the CrowdSec agent, connecting bouncers to actually block traffic, and tuning scenarios for a typical web server.

## How CrowdSec Works

CrowdSec has two components:

- **Agent** — reads log files, runs scenarios, raises "alerts" when thresholds are met
- **Bouncer** — sits at the firewall or application layer and enforces the block list

The agent never blocks anything itself. You can have multiple bouncers — firewall, Nginx, Traefik — each enforcing the same decisions.

---

## 1. Install CrowdSec Agent

```bash
# Add repository and install
curl -s https://install.crowdsec.net | sudo bash
sudo apt install -y crowdsec

# Verify
sudo cscli version
sudo systemctl status crowdsec
```

CrowdSec auto-detects running services (SSH, Nginx, Apache, MySQL) and installs relevant parsers and scenarios automatically.

---

## 2. Review Auto-Detected Configuration

```bash
# See what collections were installed
sudo cscli collections list

# See active parsers
sudo cscli parsers list

# See active scenarios (detection logic)
sudo cscli scenarios list

# See where CrowdSec is reading logs from
sudo cat /etc/crowdsec/acquis.yaml
```

A typical `acquis.yaml` entry:

```yaml
filenames:
  - /var/log/auth.log
labels:
  type: syslog
---
filenames:
  - /var/log/nginx/access.log
labels:
  type: nginx
```

---

## 3. Add More Log Sources

If CrowdSec missed a log file, add it:

```bash
sudo tee -a /etc/crowdsec/acquis.yaml > /dev/null <<'EOF'
---
filenames:
  - /var/log/nginx/error.log
labels:
  type: nginx
---
filenames:
  - /var/log/postgresql/postgresql-*.log
labels:
  type: postgresql
EOF

sudo systemctl restart crowdsec
```

---

## 4. Install Collections for Your Services

Collections bundle parsers + scenarios for a specific service:

```bash
# Common collections
sudo cscli collections install crowdsecurity/nginx
sudo cscli collections install crowdsecurity/sshd
sudo cscli collections install crowdsecurity/linux
sudo cscli collections install crowdsecurity/http-cve          # CVE scanner detection
sudo cscli collections install crowdsecurity/wordpress         # if running WP
sudo cscli collections install crowdsecurity/postfix           # if running mail

# Reload after installing collections
sudo systemctl reload crowdsec
```

---

## 5. Install the firewall Bouncer

The firewall bouncer translates CrowdSec decisions into `nftables` or `iptables` rules:

```bash
sudo apt install -y crowdsec-firewall-bouncer-nftables

# Connect to the local agent (auto-configures on install)
sudo systemctl enable --now crowdsec-firewall-bouncer

# Verify the bouncer registered
sudo cscli bouncers list
```

Test it's working:

```bash
# Manually add a test ban
sudo cscli decisions add --ip 198.51.100.1 --reason "test ban" --duration 5m

# Check it appears in nftables
sudo nft list table ip crowdsec

# Remove test ban
sudo cscli decisions delete --ip 198.51.100.1
```

---

## 6. Install the Nginx Bouncer (Optional, Deeper Inspection)

The Nginx bouncer blocks at the application layer before requests hit your app — useful for rate limiting per-endpoint:

```bash
sudo apt install -y crowdsec-nginx-bouncer

# Edit /etc/crowdsec/bouncers/crowdsec-nginx-bouncer.conf
# Set api_url = http://127.0.0.1:8080 and api_key (generated automatically)

# Add to nginx.conf in the http block:
# lua_package_path '/usr/lib/crowdsec/lua/?.lua;;';
# lua_shared_dict crowdsec_cache 50m;
# init_by_lua_block { require "crowdsec" }

sudo systemctl restart nginx
```

---

## 7. Monitor Decisions in Real Time

```bash
# Watch live alerts as they fire
sudo cscli alerts list --limit 20

# Watch decisions (actual blocks)
sudo cscli decisions list

# Filter by scenario
sudo cscli alerts list --scenario crowdsecurity/ssh-bf

# See current banned IPs with reason
sudo cscli decisions list -o human
```

Sample output:

```
 ID  │Scope:Value        │Reason                          │Action│Country│
─────┼───────────────────┼────────────────────────────────┼──────┼───────┼
 101 │Ip:203.0.113.5     │crowdsecurity/ssh-bf            │ban   │CN     │
 102 │Ip:198.51.100.22   │crowdsecurity/nginx-req-limit-ip│ban   │RU     │
```

---

## 8. Write a Custom Scenario

CrowdSec scenarios are YAML files. Here's one that fires when an IP hits 404 errors more than 15 times in 30 seconds (scanner detection):

```yaml
# /etc/crowdsec/scenarios/custom-404-scan.yaml
type: leaky
name: custom/http-404-scan
description: "Detect HTTP 404 scanners"
filter: "evt.Meta.http_status == '404'"
groupby: "evt.Meta.source_ip"
capacity: 15
leakspeed: "30s"
blackhole: 5m
labels:
  service: http
  type: scan
  remediation: true
```

```bash
sudo cscli scenarios install --file /etc/crowdsec/scenarios/custom-404-scan.yaml
sudo systemctl reload crowdsec
```

---

## 9. Enroll with the CrowdSec Console (Optional)

The cloud console gives you a dashboard and access to the community block list (CTI feed):

```bash
# Get your enrollment key from app.crowdsec.net
sudo cscli console enroll <your-enrollment-key>
sudo systemctl restart crowdsec

# This unlocks the "blocklist mirror" — subscribe to curated IP lists
# (premium) or use the free community decisions
```

Even without enrollment, CrowdSec pulls a community blocklist of known scanners and attackers via the API on startup.

---

## 10. Notifications and Alerting

```bash
# Configure Slack or email notifications
sudo cscli notifications install crowdsecurity/slack

# Edit /etc/crowdsec/notifications/slack.yaml
# Set webhook_url and min_severity (default: alert)

# Test notification
sudo cscli notifications test slack

# Assign notification to a profile
# Edit /etc/crowdsec/profiles.yaml — add 'notifications: [slack]' under relevant profile
sudo systemctl reload crowdsec
```

---

## Tuning False Positives

```bash
# Check the simulation mode — preview what decisions WOULD be made without banning
sudo cscli simulation enable crowdsecurity/nginx-req-limit-ip
sudo cscli simulation status

# Whitelist your own IP to avoid locking yourself out
sudo tee /etc/crowdsec/parsers/s02-enrich/whitelist.yaml > /dev/null <<'EOF'
name: my/whitelist
description: "Whitelist office and monitoring IPs"
whitelist:
  reason: "Internal IPs"
  ip:
    - "203.0.113.10"
    - "10.0.0.0/8"
EOF

sudo systemctl reload crowdsec
```

---

## Metrics

```bash
# Show agent metrics: alerts fired, scenarios triggered
sudo cscli metrics

# Prometheus endpoint (if enabled in config.yaml)
# metrics_addr: 127.0.0.1:6060
curl http://127.0.0.1:6060/metrics | grep crowdsec
```

---

## Related Reading

- [How to Set Up Snort IDS on Linux](/privacy-tools-guide/snort-ids-linux-setup-guide/)
- [How to Set Up Wazuh SIEM for Small Teams](/privacy-tools-guide/wazuh-siem-small-teams-setup-guide/)
- [How to Set Up ModSecurity WAF with Nginx](/privacy-tools-guide/modsecurity-waf-nginx-setup-guide/)
- [How to Set Up age Encryption for Teams](/privacy-tools-guide/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
