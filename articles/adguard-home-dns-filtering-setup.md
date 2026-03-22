---
layout: default
title: "Privacy-Focused DNS Filtering with AdGuard Home"
description: "Install AdGuard Home on your server or Raspberry Pi, configure blocklists, DNS-over-HTTPS, and per-client rules for network-wide ad and tracker blocking"
date: 2026-03-22
author: theluckystrike
permalink: /adguard-home-dns-filtering-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Privacy-Focused DNS Filtering with AdGuard Home

AdGuard Home is a self-hosted DNS server that blocks ads, trackers, and malware domains before they reach any device on your network. Unlike Pi-hole, it ships with built-in DNS-over-HTTPS and DNS-over-TLS, per-client rules, and a query log with filtering — no separate proxy required.

## Key Takeaways

- **Topics covered**: installation, initial configuration via web ui, configure encrypted upstream dns
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## Table of Contents

- [Installation](#installation)
- [Initial Configuration via Web UI](#initial-configuration-via-web-ui)
- [Configure Encrypted Upstream DNS](#configure-encrypted-upstream-dns)
- [Blocklists](#blocklists)
- [Per-Client Rules](#per-client-rules)
- [Custom Rules (Allowlist/Blocklist)](#custom-rules-allowlistblocklist)
- [DNS Rewrites for Local Services](#dns-rewrites-for-local-services)
- [Point Your Router to AdGuard Home](#point-your-router-to-adguard-home)
- [Setting Up DNS-over-HTTPS for Remote Devices](#setting-up-dns-over-https-for-remote-devices)
- [Query Log Analysis](#query-log-analysis)
- [Statistics Export](#statistics-export)
- [Advanced Filtering Rules and Regex](#advanced-filtering-rules-and-regex)
- [Detecting Over-Blocking Issues](#detecting-over-blocking-issues)
- [Multi-Device Client Rules Setup](#multi-device-client-rules-setup)
- [Monitoring Query Patterns](#monitoring-query-patterns)
- [Backup and Restore Strategy](#backup-and-restore-strategy)
- [Performance Tuning](#performance-tuning)
- [Related Reading](#related-reading)

## Installation

```bash
# Install on Linux (x86_64)
curl -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh

# Or manual install
wget https://github.com/AdguardTeam/AdGuardHome/releases/latest/download/AdGuardHome_linux_amd64.tar.gz
tar xzf AdGuardHome_linux_amd64.tar.gz
cd AdGuardHome
sudo ./AdGuardHome -s install

# Check service status
sudo systemctl status AdGuardHome
```

For Raspberry Pi (ARM):

```bash
wget https://github.com/AdguardTeam/AdGuardHome/releases/latest/download/AdGuardHome_linux_arm64.tar.gz
tar xzf AdGuardHome_linux_arm64.tar.gz
cd AdGuardHome
sudo ./AdGuardHome -s install
```

Open the web UI at `http://your-server-ip:3000` for initial setup.

## Initial Configuration via Web UI

During setup, choose:
- **Listen interface:** Your LAN interface (e.g., `eth0` or `wlan0`) — not `0.0.0.0` in production
- **Admin UI port:** Change from 3000 to 3001 or 8080
- **DNS port:** 53 (default)
- **DNS upstreams:** Configure in the next section

## Configure Encrypted Upstream DNS

In **Settings → DNS settings → Upstream DNS servers**, replace the defaults:

```
# DNS-over-HTTPS options (add one or more)
https://dns10.quad9.net/dns-query
https://cloudflare-dns.com/dns-query
https://dns.mullvad.net/dns-query

# DNS-over-TLS options
tls://dns10.quad9.net
tls://1dot1dot1dot1.cloudflare-dns.com

# Bootstrap DNS (used to resolve the DoH hostname itself)
# Use plain DNS only for bootstrap
9.9.9.9
1.1.1.1
```

Enable **Parallel requests** to query multiple upstreams simultaneously and use the fastest response.

Under **Fallback DNS servers**, set a reliable plain-DNS fallback in case DoH fails:

```
9.9.9.9
149.112.112.112
```

## Blocklists

In **Filters → DNS blocklists → Add blocklist**, add:

| List | URL | Blocks |
|------|-----|--------|
| AdGuard DNS Filter | https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt | Ads, trackers |
| EasyList | https://easylist.to/easylist/easylist.txt | Ads |
| EasyPrivacy | https://easylist.to/easylist/easyprivacy.txt | Trackers |
| HaGeZi Pro | https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt | Combined |
| OISD | https://abp.oisd.nl/basic | Ads, malware |
| Malware Domain List | https://www.malwaredomainlist.com/hostslist/hosts.txt | Malware |

After adding, click **Update** and check the block count — a healthy setup blocks 200,000–500,000 domains.

## Per-Client Rules

AdGuard Home can apply different rules to different devices. In **Settings → Client settings → Add client**:

```
Client: Kids iPad
  Identifier: 192.168.1.45 (or MAC: AA:BB:CC:DD:EE:FF)
  Extra blocklists:
    - https://raw.githubusercontent.com/nicehash/NiceHash/master/lists/adult-blocklist.txt
  SafeSearch: Force enabled
  Blocked services: [Gaming] [Social Media]
```

```
Client: Work Laptop
  Identifier: 192.168.1.10
  Upstream DNS: https://dns.cloudflare.com/dns-query  (bypass custom lists if needed)
  Block: disabled  (or minimal)
```

## Custom Rules (Allowlist/Blocklist)

In **Filters → Custom filtering rules**:

```
# Block a specific domain
||analytics.example.com^

# Block all subdomains of a tracker
||doubleclick.net^

# Allow a domain that a blocklist is falsely blocking
@@||cdn.legitimate-site.com^

# Block all traffic to a specific IP
|185.234.219.0/24

# Rewrite DNS for local services (A record override)
local.myapp.internal A 192.168.1.20
```

## DNS Rewrites for Local Services

In **Filters → DNS rewrites**, map local hostnames:

```
homeassistant.local → 192.168.1.50
pihole.local → 192.168.1.51
nas.local → 192.168.1.100
```

This lets you access local services by name without managing `/etc/hosts` on every device.

## Point Your Router to AdGuard Home

For network-wide coverage, set your router's DNS to your AdGuard Home server IP.

**On most home routers:** DHCP settings → Primary DNS: `192.168.1.x` (your server)

```bash
# Verify devices are using your DNS
# On a device on the network:
nslookup doubleclick.net
# Should return 0.0.0.0 (blocked) instead of the real IP
```

If you cannot change router DNS, configure each device individually:
- **Android 9+:** Private DNS → custom → enter your server's hostname or IP
- **iOS:** Install a DNS profile or use an app like DNSCloak
- **Windows:** Network adapter DNS settings

## Setting Up DNS-over-HTTPS for Remote Devices

AdGuard Home can serve DoH to devices outside your network:

```bash
# AdGuard Home config: /opt/AdGuardHome/AdGuardHome.yaml
tls:
  enabled: true
  server_name: "dns.yourdomain.com"
  force_https: true
  port_https: 443
  port_dns_over_tls: 853
  port_dns_over_quic: 784
  certificate_path: "/etc/letsencrypt/live/dns.yourdomain.com/fullchain.pem"
  private_key_path: "/etc/letsencrypt/live/dns.yourdomain.com/privkey.pem"
```

```bash
# Get a certificate with certbot
sudo certbot certonly --standalone -d dns.yourdomain.com

# Restart AdGuard Home
sudo systemctl restart AdGuardHome
```

Remote devices connect to `https://dns.yourdomain.com/dns-query`.

## Query Log Analysis

The query log shows every DNS request, which client made it, and whether it was blocked.

```bash
# Query log is stored at (default):
/opt/AdGuardHome/data/querylog.json

# Count top blocked domains
jq -r 'select(.Result.IsFiltered == true) | .QH' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -20

# Count queries per client
jq -r '.IP' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -10

# Find clients that bypass your DNS (query from unexpected IPs)
```

## Statistics Export

```bash
# Export stats via API
curl http://localhost:3000/control/stats \
  -H "Authorization: Basic $(echo -n 'admin:yourpassword' | base64)" | jq '{
    total_queries: .num_dns_queries,
    blocked: .num_blocked_filtering,
    block_ratio: (.num_blocked_filtering / .num_dns_queries * 100 | round)
  }'
```

## Advanced Filtering Rules and Regex

Beyond simple blocklists, create powerful custom rules:

```
# AdGuard Home custom filtering rules - Filters tab

# Block all analytics services
||analytics.google.com^
||segment.com^
||amplitude.com^
||mixpanel.com^

# Block all third-party tracking pixels
||doubleclick.net^
||ads.yahoo.com^
||tracking.kenshoo.com^

# Advanced: Block using regex (treat domain as regex)
/.+\.facebook\.com^

# Allow specific domain that's overly blocked
@@||cdn.example-legitimate-site.com^

# Block IP ranges (blocking ads from specific networks)
|185.93.187.0/24

# DNS rewrite (map fake domain to localhost for testing)
||test.internal^ 127.0.0.1

# Case-sensitive matching
||MyService.com^

# Exact match only (don't match subdomains)
|exact.domain.com^

# Comments (start with #)
# This blocks all Adobe analytics
||adobe.analytics.com^
```

## Detecting Over-Blocking Issues

Identify which blocklist is causing false positives:

```bash
#!/bin/bash
# Debug script to find problematic blocklist

BROKEN_DOMAIN="service-that-should-work.com"
CURL_TIMEOUT=5

echo "Testing: $BROKEN_DOMAIN"
curl -s --max-time $CURL_TIMEOUT $BROKEN_DOMAIN

if [ $? -ne 0 ]; then
  echo "ERROR: Connection failed or timed out"
  echo "Checking which blocklist is blocking this..."

  # Query AdGuard Home debug API
  curl -s "http://localhost:3000/control/filtering/check?name=$BROKEN_DOMAIN" \
    -H "Authorization: Basic $(echo -n 'admin:password' | base64)" | jq '.rules'

  # Output will show which filter(s) are blocking
fi
```

## Multi-Device Client Rules Setup

Configure different filtering for different device types:

```
# Kids devices - strict filtering
Client: Tablet-Kids
  Identifier: 192.168.1.45
  Extra blocklists:
    - https://raw.githubusercontent.com/nicehash/NiceHash/master/lists/adult-blocklist.txt
  SafeSearch: Enabled (all search engines)
  Safe Browsing: Enabled
  Blocked services: [Gaming, Social Media, YouTube]
  Parental control labels: Safe content only

# Work device - minimal filtering
Client: Work-Laptop
  Identifier: 192.168.1.10
  Blocklists: Disabled (or minimal)
  Upstream DNS: https://dns.cloudflare.com/dns-query
  Block: Disabled
  Purpose: All filtering would interfere with business tools

# Guest network - moderate filtering
Client: Guest-WiFi
  Identifier: 192.168.1.200/24 (subnet for all guests)
  Blocklists: Standard (ads/malware only)
  Safe Search: Enabled
  Rate limiting: Enabled (prevent abuse)

# IoT devices - strict to prevent data exfiltration
Client: Smart-Home
  Identifier: 192.168.1.50-60 (all IoT devices)
  Blocklists: EasyPrivacy + HaGeZi + OISD (aggressive tracking blocking)
  Upstream DNS: Quad9 (malware filtering)
  Custom rules: Block all outbound except to approved IPs
```

## Monitoring Query Patterns

Analyze what's being blocked to optimize filtering:

```bash
#!/bin/bash
# analyze-queries.sh - Find patterns in DNS blocking

# Top blocked domains
echo "=== Top 20 Blocked Domains ==="
jq -r 'select(.Result.IsFiltered == true) | .QH' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -20

# Percentage of queries blocked
echo "=== Blocking Statistics ==="
TOTAL=$(jq 'length' /opt/AdGuardHome/data/querylog.json)
BLOCKED=$(jq 'select(.Result.IsFiltered == true) | .QH' /opt/AdGuardHome/data/querylog.json | wc -l)
PERCENTAGE=$((BLOCKED * 100 / TOTAL))
echo "Blocked: $BLOCKED out of $TOTAL ($PERCENTAGE%)"

# Queries by client (who's being filtered most)
echo "=== Top Clients by Query Count ==="
jq -r '.IP' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -10

# Failed/timed out queries (potential issues)
echo "=== Failed Queries ==="
jq -r 'select(.Result.Reason == "NotFiltered" or .Result.Reason == "Blocked") | .QH' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -10
```

## Backup and Restore Strategy

Regular backups protect against accidental misconfiguration:

```bash
#!/bin/bash
# backup-adguard.sh - Regular AdGuard Home backup

BACKUP_DIR="/backup/adguard"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Backup configuration
cp -r /opt/AdGuardHome/data "$BACKUP_DIR/data-$TIMESTAMP"

# Backup database
cp /opt/AdGuardHome/AdGuardHome.db "$BACKUP_DIR/AdGuardHome-$TIMESTAMP.db"

# Export settings as JSON via API
curl -s http://localhost:3000/control/config \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  > "$BACKUP_DIR/config-$TIMESTAMP.json"

# Keep only last 30 days of backups
find "$BACKUP_DIR" -name "data-*" -mtime +30 -exec rm -rf {} \;
find "$BACKUP_DIR" -name "*.db" -mtime +30 -exec rm {} \;

# Encrypt backups
tar czf - "$BACKUP_DIR" | openssl enc -aes-256-cbc -out "$BACKUP_DIR/backup-$TIMESTAMP.tar.gz.enc"

echo "Backup complete: $BACKUP_DIR/backup-$TIMESTAMP.tar.gz.enc"
```

## Performance Tuning

Optimize AdGuard Home for high traffic:

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

# Increase query cache for faster responses
cache:
  size: 100000  # From default 10000
  ttl_min: 60   # Minimum cache time

# Connection pooling
upstream_dns_pool:
  size: 8       # Parallel upstream connections

# Rate limiting to prevent abuse
ratelimit:
  enabled: true
  clients_limit: 1000    # Max clients
  whitelisted: []        # Whitelist trusted IPs

# Query log size (balance between detail and disk usage)
querylog:
  enabled: true
  file_size: 100  # MB before rotation
  interval: 24h   # Rotation interval
  anonymize_client_ip: true  # Privacy

# Performance tweaking
upstream_mode: fastest  # Use fastest upstream, not load-balanced
```

## Related Reading

- [Home Network Privacy: Pi-hole DNS Filtering Guide](/privacy-tools-guide/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-tools-guide/privacy-dns-resolver-comparison-2026/)
- [How to Configure DNS over HTTPS for Privacy](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/)

---

## Related Articles

- [Home Network Privacy Pihole Dns Filtering Guide 2026](/privacy-tools-guide/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [Privacy-Focused DNS Providers Comparison 2026: Privacy](/privacy-tools-guide/privacy-focused-dns-providers-comparison/)
- [Privacy-Focused DNS Providers Comparison 2026](/privacy-tools-guide/privacy-focused-dns-providers-comparison-2026/)
- [Privacy-Focused DNS over QUIC Setup](/privacy-tools-guide/dns-over-quic-setup-guide/)
- [How to Set Up a Privacy Focused Home](/privacy-tools-guide/how-to-set-up-a-privacy-focused-home-network/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
