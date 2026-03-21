---
layout: default
title: "Encrypted DNS over HTTPS on Linux"
description: "Configure DNS over HTTPS on Linux using systemd-resolved, dnscrypt-proxy, or AdGuard Home. Stop your ISP from logging every domain you visit"
date: 2026-03-21
author: theluckystrike
permalink: encrypted-dns-over-https-linux-setup
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

By default, DNS queries on Linux go to your ISP's resolver in plaintext on UDP port 53. Anyone between you and that resolver — your ISP, your network admin, a rogue router — can log every domain you visit. DNS over HTTPS (DoH) encrypts those queries inside standard HTTPS traffic on port 443.

This guide covers three approaches: the built-in `systemd-resolved` (no extra software), `dnscrypt-proxy` (flexible, supports filters), and `AdGuard Home` (self-hosted with a web UI).

## Method 1: systemd-resolved with DoH

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

## Method 2: dnscrypt-proxy

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

## Method 3: AdGuard Home (Self-Hosted)

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

## Choosing a DoH Provider

| Provider | Logs | DNSSEC | Filtering | Notes |
|----------|------|--------|-----------|-------|
| Cloudflare (1.1.1.1) | No query logs | Yes | Optional | Fast, widely audited |
| Quad9 (9.9.9.9) | No PII | Yes | Malware blocking | Swiss nonprofit |
| NextDNS | Configurable | Yes | Highly configurable | Free tier available |
| AdGuard DNS | Minimal | Yes | Ad/tracker blocking | Self-hostable |

Avoid using your ISP's or Google's DNS for privacy — they log queries.

## Verify the Full Setup

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

## Related Reading

- [Privacy-Focused DNS Providers Comparison](/privacy-focused-dns-providers-comparison)
- [How to Configure DNS over HTTPS Inside VPN Tunnel](/how-to-configure-dns-over-https-inside-vpn-tunnel)
- [How to Set Up WireGuard on VPS for Personal VPN](/how-to-set-up-wireguard-on-vps-for-personal-vpn)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
