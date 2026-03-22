---
layout: default
title: "Privacy-Focused DNS Filtering with AdGuard Home"
description: "Install AdGuard Home on your server or Raspberry Pi, configure blocklists, DNS-over-HTTPS, and per-client rules for network-wide ad and tracker blocking"
date: 2026-03-22
author: theluckystrike
permalink: /adguard-home-dns-filtering-setup/
categories: [guides, privacy]
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

## Related Reading

- [Home Network Privacy: Pi-hole DNS Filtering Guide](/privacy-tools-guide/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-tools-guide/privacy-dns-resolver-comparison-2026/)
- [How to Configure DNS over HTTPS for Privacy](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
