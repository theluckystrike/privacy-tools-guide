---
layout: default
title: "How to Set Up ntopng for Network Analytics"
description: "Install and configure ntopng to monitor real-time network traffic, detect anomalies, and identify privacy-leaking hosts on your network"
date: 2026-03-22
author: theluckystrike
permalink: /ntopng-network-analytics-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

How to Set Up ntopng for Network Analytics

ntopng is a high-speed network traffic analysis tool that runs in your browser. It shows you exactly which hosts are talking to what, how much bandwidth each device consumes, and which connections look suspicious. This guide walks you through a complete setup on Ubuntu/Debian.

Prerequisites

- Ubuntu 22.04 or Debian 12
- Root or sudo access
- At least 2 GB RAM (4 GB recommended for active networks)
- A mirrored network port or inline placement on your router/switch

Step 1. Install ntopng

ntop provides its own APT repository with the latest stable builds.

```bash
Install dependencies
sudo apt update && sudo apt install -y wget gnupg

Add ntop repository
wget https://packages.ntop.org/apt-stable/22.04/all/apt-ntop-stable.deb
sudo apt install ./apt-ntop-stable.deb

Install ntopng and nProbe lite
sudo apt update && sudo apt install -y ntopng
```

Verify the install:

```bash
ntopng --version
ntopng 6.x. community edition
```

Step 2. Configure ntopng

The main config lives at `/etc/ntopng/ntopng.conf`. Edit it:

```bash
sudo nano /etc/ntopng/ntopng.conf
```

Paste in a working baseline:

```ini
Interface to monitor. replace with your interface name
-i=eth0

HTTP server port
--http-port=3000

Community license. no registration needed for basic use
--community

Store data in a persistent directory
-d=/var/lib/ntopng

Disable login for LAN-only access (remove in production)
--disable-login=1

Set local network range so ntopng knows your LAN
--local-networks=192.168.1.0/24

DNS resolution for hostnames in the UI
--dns-mode=1

Redis connection (ntopng uses Redis for caching)
--redis=127.0.0.1:6379
```

Find your interface name:

```bash
ip link show
Look for: eth0, enp3s0, wlan0, etc.
```

Step 3. Install and Start Redis

ntopng requires Redis for its internal data cache:

```bash
sudo apt install -y redis-server
sudo systemctl enable --now redis-server

Confirm Redis is listening
redis-cli ping
PONG
```

Step 4. Start ntopng

```bash
sudo systemctl enable --now ntopng
sudo systemctl status ntopng
```

Access the dashboard at `http://localhost:3000`. Default credentials are `admin` / `admin`. change these immediately under Settings > Manage Users.

Step 5. Configure Traffic Mirroring

ntopng needs to see traffic from all devices, not just the machine it runs on. On a managed switch, enable port mirroring (SPAN) to copy all traffic to the port your ntopng machine is connected to.

On a home router running OpenWrt:

```bash
SSH into OpenWrt router
Mirror LAN traffic to a monitoring port using tc
tc qdisc add dev br-lan handle ffff: ingress
tc filter add dev br-lan parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev eth3
```

Replace `eth3` with the port connected to your ntopng machine.

Step 6. Set Up Alerts for Suspicious Hosts

Navigate to Alerts > Preferences in the ntopng UI. Enable:

- Too many alerts. catches port scanners
- Blacklisted host contacted. flags known malware C2 domains
- Suspicious DNS queries. detects DNS tunneling or data exfiltration
- New device detected. alerts on rogue devices joining your network

Enable blacklists under Settings > Blacklists. ntopng ships with several threat intelligence feeds you can activate in one click.

Step 7. Harden ntopng Access

The web interface should not be exposed to the internet. Lock it down:

```bash
Allow only LAN access to port 3000
sudo ufw allow from 192.168.1.0/24 to any port 3000
sudo ufw deny 3000

Reload firewall
sudo ufw reload
```

Re-enable login authentication in `/etc/ntopng/ntopng.conf` by removing or commenting out `--disable-login=1`, then restart:

```bash
sudo systemctl restart ntopng
```

Step 8. Enable HTTPS for the Dashboard

Generate a self-signed certificate for local use:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout /etc/ntopng/server.key \
  -out /etc/ntopng/server.crt \
  -subj "/CN=ntopng-local"
```

Add to `/etc/ntopng/ntopng.conf`:

```ini
--https-port=3001
--ssl-private-key=/etc/ntopng/server.key
--ssl-certificate=/etc/ntopng/server.crt
```

Restart and access via `https://localhost:3001`.

Step 9. Query Traffic Data via the REST API

ntopng exposes a REST API you can query for automated monitoring:

```bash
Get current top talkers
curl -s -u admin:yourpassword \
  "http://localhost:3000/lua/rest/v2/get/host/top_sites.lua?ifid=1" | jq .

List all active hosts
curl -s -u admin:yourpassword \
  "http://localhost:3000/lua/rest/v2/get/interface/hosts.lua?ifid=1" | jq .hosts[]
```

Use these in cron jobs or incident response scripts to pull traffic snapshots on demand.

Reading the Dashboard

Hosts Map. shows geolocation of external connections. Any unexpected country should raise a flag.

Flows. live list of every active TCP/UDP session. Filter by host or port to investigate specific traffic.

Applications. protocol breakdown. High BitTorrent or Tor traffic may indicate unauthorized usage or a compromised device.

Historical Data. ntopng stores per-host traffic graphs. Use this to establish baselines and spot deviations.

Identifying Privacy-Leaking Traffic

Look for:
- Devices talking to telemetry endpoints (e.g., `*.googleapis.com`, `*.microsoft.com`, `*.apple.com`) without user interaction
- Smart home devices phoning home to vendor cloud servers
- DNS queries to unexpected resolvers. a sign of DNS over HTTPS bypassing your Pi-hole
- Long-lived connections on unusual ports. possible exfiltration or C2 beacons

Related Reading

- [How to Set Up Pi-hole for DNS-Level Ad Blocking](/pihole-setup-guide/)
- [How to Use tcpdump for Packet Analysis](/tcpdump-packet-analysis-guide/)
- [VPN Kill Switch with iptables on Linux](/vpn-kill-switch-linux-iptables-setup/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
