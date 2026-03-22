---
layout: default
title: "Encrypted DNS over HTTPS on Linux"
description: "Configure DNS over HTTPS on Linux using systemd-resolved, dnscrypt-proxy, or AdGuard Home. Stop your ISP from logging every domain you visit"
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: encrypted-dns-over-https-linux-setup
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

By default, DNS queries on Linux go to your ISP's resolver in plaintext on UDP port 53. Anyone between you and that resolver — your ISP, your network admin, a rogue router — can log every domain you visit. DNS over HTTPS (DoH) encrypts those queries inside standard HTTPS traffic on port 443.

This guide covers three approaches: the built-in `systemd-resolved` (no extra software), `dnscrypt-proxy` (flexible, supports filters), and `AdGuard Home` (self-hosted with a web UI).

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Method 1: systemd-resolved with DoH

Most modern Debian/Ubuntu and Fedora systems already use `systemd-resolved`. Check:

```bash
systemctl status systemd-resolved
resolvectl status | head -20
```

### Enable DNS over HTTPS in systemd-resolved

Edit the resolved configuration:

```bash
sudo nano /etc/systemd/resolved.conf
```

Add or modify:

```ini
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 9.9.9.9#dns.quad9.net
FallbackDNS=8.8.8.8#dns.google
DNSSEC=yes
DNSOverTLS=yes
```

`DNS=` accepts `IP#hostname` format. The `#hostname` part is used to verify the TLS certificate.

Restart resolved:

```bash
sudo systemctl restart systemd-resolved
```

Verify DoH is active:

```bash
resolvectl status | grep -E "(DNS Server|DNS over TLS|DNSSEC)"
```

Expected output:
```
Current DNS Server: 1.1.1.1#cloudflare-dns.com
DNS over TLS setting: yes
DNSSEC setting: yes
```

Test a lookup and confirm it works:

```bash
resolvectl query example.com
```

### Confirm No Plaintext DNS Leaks

Capture traffic to check that port 53 traffic is gone:

```bash
sudo tcpdump -i any port 53 -c 20 &
dig example.com
```

You should see no port 53 traffic if DoH is working correctly. All queries should appear on port 443.

### Step 2: Method 2: dnscrypt-proxy

`dnscrypt-proxy` supports DoH, DNSCrypt, and ODoH (Oblivious DoH). It runs as a local resolver on `127.0.0.1:53` and proxies queries through encrypted channels.

### Install dnscrypt-proxy

```bash
# Debian/Ubuntu
sudo apt install dnscrypt-proxy

# Fedora
sudo dnf install dnscrypt-proxy

# Arch
sudo pacman -S dnscrypt-proxy
```

### Configure dnscrypt-proxy

```bash
sudo nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

Key settings to configure:

```toml
# Listen on localhost port 53
listen_addresses = ['127.0.0.1:53', '[::1]:53']

# Use only DoH servers (not plain DNS)
server_names = ['cloudflare', 'quad9-doh-ip4-filter-pri', 'nextdns']

# Filter out servers without DNSSEC
require_dnssec = true

# Filter out servers that log queries
require_nolog = true

# Filter out servers that don't require NOFILTER
require_nofilter = false

# Block ads and trackers with built-in blocklists
[sources]
  [sources.public-resolvers]
  urls = ['https://raw.githubusercontent.com/DNSCrypt/dnscrypt-resolvers/master/v3/public-resolvers.md']
  cache_file = '/var/cache/dnscrypt-proxy/public-resolvers.md'
  minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
  refresh_delay = 72

# Enable CLOAKING for local overrides (optional)
# [cloaking]
# cloaking_rules = '/etc/dnscrypt-proxy/cloaking-rules.txt'

# Block known malware/ad domains
[blocked_names]
blocked_names_file = '/etc/dnscrypt-proxy/blocked-names.txt'
```

Start and enable the service:

```bash
sudo systemctl enable --now dnscrypt-proxy
```

### Point systemd-resolved at dnscrypt-proxy

Edit resolved.conf to use the local proxy:

```bash
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
DNS=127.0.0.1
DNSOverTLS=no
DNSSEC=no
```

(dnscrypt-proxy handles DNSSEC and encryption itself.)

Restart both services:

```bash
sudo systemctl restart dnscrypt-proxy systemd-resolved
```

Test:

```bash
dig @127.0.0.1 example.com
```

Check which server is actually being used:

```bash
dnscrypt-proxy -resolve example.com
```

### Step 3: Method 3: AdGuard Home (Self-Hosted)

AdGuard Home is a DNS server with DoH support, ad blocking, and a web interface. Good for households or small networks.

### Install AdGuard Home

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

The installer places the binary in `/opt/AdGuardHome/`. Access the setup wizard at `http://localhost:3000`.

During setup:
- Set the DNS server to listen on port 53
- Set the admin interface to port 3000 (or another port)
- Choose upstream DNS servers with DoH enabled

### Configure DoH Upstream in AdGuard Home

In the web UI, go to **Settings > DNS settings > Upstream DNS servers**:

```
https://dns.cloudflare.com/dns-query
https://dns.quad9.net/dns-query
https://dns.adguard-dns.com/dns-query
```

Enable **Parallel requests** for faster lookups.

Set your system DNS to `127.0.0.1`:

```bash
sudo resolvectl dns eth0 127.0.0.1
```

Or edit `/etc/resolv.conf` directly (if not managed by systemd-resolved):

```bash
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

Lock the file against NetworkManager overwrites:

```bash
sudo chattr +i /etc/resolv.conf
```

### Step 4: Choose a DoH Provider

| Provider | Logs | DNSSEC | Filtering | Notes |
|----------|------|--------|-----------|-------|
| Cloudflare (1.1.1.1) | No query logs | Yes | Optional | Fast, widely audited |
| Quad9 (9.9.9.9) | No PII | Yes | Malware blocking | Swiss nonprofit |
| NextDNS | Configurable | Yes | Highly configurable | Free tier available |
| AdGuard DNS | Minimal | Yes | Ad/tracker blocking | Self-hostable |

Avoid using your ISP's or Google's DNS for privacy — they log queries.

### Step 5: Verify the Full Setup

Check DNS resolution is working:

```bash
nslookup example.com
curl -s "https://www.dnsleaktest.com/results.json" | python3 -m json.tool | grep name
```

Run a DNS leak test at `https://dnsleaktest.com` — all results should show your chosen provider, not your ISP.

Confirm DNSSEC validation:

```bash
dig +dnssec sigfail.verteiltesysteme.net
# Should return SERVFAIL if DNSSEC validation is working
```

## Troubleshooting Common Issues

### DNS Resolution Fails After Configuration

If you lose DNS resolution after enabling DoH, the most likely cause is a configuration syntax error:

```bash
# Temporary fix: use a direct DNS server
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf

# Then review your configuration files for typos
sudo journalctl -u systemd-resolved --no-pager -n 50
```

### Slow DNS Lookups

DoH adds TLS overhead compared to plain DNS. If lookups feel slow, check latency:

```bash
time dig @1.1.1.1 example.com
time dig @9.9.9.9 example.com
time dig @8.8.8.8 example.com
```

Choose the provider with the lowest latency from your location. Cloudflare consistently provides the fastest response times due to its CDN presence.

### NetworkManager Overwriting resolv.conf

NetworkManager frequently overwrites `/etc/resolv.conf`. Prevent this:

```bash
# /etc/NetworkManager/conf.d/dns.conf
[main]
dns=systemd-resolved
```

Then restart: `sudo systemctl restart NetworkManager`

## Performance Comparison: DoH vs Standard DNS

| Metric | Standard DNS (UDP 53) | DNS over TLS | DNS over HTTPS |
|--------|----------------------|--------------|----------------|
| First lookup latency | ~20ms | ~80ms | ~100ms |
| Cached lookup latency | ~1ms | ~1ms | ~1ms |
| Privacy from ISP | None | Full | Full |
| Resistance to blocking | None | Moderate (port 853) | High (port 443) |
| CPU overhead | Minimal | Low | Low-Medium |

The first lookup is slower with DoH due to the TLS handshake, but subsequent lookups use cached connections and perform comparably.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [How to Set Up Encrypted DNS-over-HTTPS (DoH) on All Devices](/privacy-tools-guide/how-to-set-up-encrypted-dns-over-https-doh-on-all-devices-guide/)
- [How to Configure DNS over HTTPS Inside a VPN Tunnel](/privacy-tools-guide/how-to-configure-dns-over-https-inside-vpn-tunnel/)
- [Linux Mint Privacy Setup Guide for Beginners](/privacy-tools-guide/linux-mint-privacy-setup-guide-beginners/)
- [Linux Secure Boot Setup with Custom Keys for Preventing.](/privacy-tools-guide/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [Proton Drive Linux Client Setup Guide 2026](/privacy-tools-guide/proton-drive-linux-client-setup-guide-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
