---
layout: default
title: "Privacy-Focused DNS over QUIC Setup"
description: "Configure DNS over QUIC (DoQ) on Linux, Android, and routers for faster encrypted DNS with lower latency than DoT and stronger privacy than plain DNS"
date: 2026-03-22
author: theluckystrike
permalink: /dns-over-quic-setup-guide/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Privacy-Focused DNS over QUIC Setup

DNS over QUIC (DoQ, RFC 9250) is the newest encrypted DNS protocol. It uses QUIC transport — the same protocol behind HTTP/3 — which means lower connection setup latency than DNS over TLS (DoT) and better performance on unreliable networks. DoQ also prevents DNS traffic from being identified and blocked by DPI (Deep Packet Inspection) as easily as DoT.

## Key Takeaways

- **It uses QUIC transport**: the same protocol behind HTTP/3 — which means lower connection setup latency than DNS over TLS (DoT) and better performance on unreliable networks.
- **Topics covered**: why doq over dot or doh, public doq resolvers, configuration on linux (adguard home)
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Why DoQ Over DoT or DoH

| Protocol | Transport | Port | Latency | Fingerprint risk |
|----------|-----------|------|---------|-----------------|
| DNS over TLS (DoT) | TLS/TCP | 853 | Higher (TCP handshake) | Easily blocked on port 853 |
| DNS over HTTPS (DoH) | TLS/TCP | 443 | Moderate | Looks like HTTPS |
| DNS over QUIC (DoQ) | QUIC/UDP | 853 | Lower (0-RTT) | Harder to distinguish from QUIC traffic |

DoQ runs on UDP, which means no TCP three-way handshake, and QUIC has built-in connection migration — DNS queries complete even if your network changes mid-request.

## Public DoQ Resolvers

| Provider | DoQ address | Policy |
|----------|------------|--------|
| Mullvad | `quic://dns.mullvad.net` | No logs, no AAAA filtering |
| AdGuard | `quic://dns.adguard-dns.com` | Ad blocking built in |
| NextDNS | `quic://YOUR-ID.dns.nextdns.io` | Per-account config |
| Quad9 | `quic://dns.quad9.net` | Malware blocking, no logs |

## Configuration on Linux (AdGuard Home)

AdGuard Home supports DoQ as both a client (upstream) and server:

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

# Use DoQ as upstream
upstream_dns:
  - quic://dns.mullvad.net
  - quic://dns.adguard-dns.com

# Fallback if DoQ fails
fallback_dns:
  - 9.9.9.9

# Serve DoQ to your clients (requires TLS certificate)
tls:
  enabled: true
  server_name: dns.yourdomain.com
  port_dns_over_quic: 784     # RFC 9250 registered port
  port_dns_over_tls: 853
  certificate_path: /etc/letsencrypt/live/dns.yourdomain.com/fullchain.pem
  private_key_path: /etc/letsencrypt/live/dns.yourdomain.com/privkey.pem
```

```bash
# Verify DoQ is listening
ss -ulnp | grep 784
# UNCONN  0  0  0.0.0.0:784  0.0.0.0:*  users:(("AdGuardHome",pid=1234))
```

## Configuration on Linux (dnsproxy)

dnsproxy is a lightweight proxy from AdGuard that supports all encrypted DNS protocols:

```bash
# Install dnsproxy
wget https://github.com/AdguardTeam/dnsproxy/releases/latest/download/dnsproxy-linux-amd64-*.tar.gz
tar xzf dnsproxy-linux-amd64-*.tar.gz
sudo mv dnsproxy /usr/local/bin/

# Run with DoQ upstream, listen locally for all protocols
sudo dnsproxy \
  --listen=127.0.0.1 \
  --port=53 \
  --upstream="quic://dns.mullvad.net" \
  --fallback="https://9.9.9.9/dns-query" \
  --tls-port=853 \
  --tls-crt=/etc/letsencrypt/live/dns.yourdomain.com/fullchain.pem \
  --tls-key=/etc/letsencrypt/live/dns.yourdomain.com/privkey.pem \
  --quic-port=784 \
  --cache \
  --cache-size=2048
```

```ini
# /etc/systemd/system/dnsproxy.service
[Unit]
Description=DNS Proxy (DoQ)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/dnsproxy \
  --listen=127.0.0.1 \
  --port=5353 \
  --upstream=quic://dns.mullvad.net \
  --fallback=https://9.9.9.9/dns-query \
  --cache \
  --cache-size=2048
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now dnsproxy

# Point local resolver to dnsproxy
sudo tee /etc/resolv.conf << 'EOF'
nameserver 127.0.0.1
options edns0 trust-ad
EOF
```

## Testing DoQ with q (Command-Line DNS Client)

```bash
# Install q — a modern DNS client with DoQ support
go install github.com/natesales/q@latest

# Query over DoQ
q example.com A @quic://dns.mullvad.net
q example.com AAAA @quic://dns.adguard-dns.com

# Compare latency: DoQ vs DoT vs DoH vs plain DNS
time q example.com @quic://dns.mullvad.net
time q example.com @tls://dns.mullvad.net
time q example.com @https://dns.mullvad.net/dns-query
time q example.com @1.1.1.1
```

```bash
# Or use doggo (another modern DNS CLI)
brew install doggo   # macOS
# Linux: download from github.com/mr-karan/doggo/releases

doggo example.com --type A @quic://dns.mullvad.net
```

## Android Configuration

Android 9+ supports Private DNS (DoT). DoQ on Android requires a third-party app:

- **Intra** by Jigsaw — supports DoH and DoQ
- **DNSCloak** — supports all protocols including DoQ
- **AdGuard for Android** — supports DoQ natively

```
In AdGuard for Android:
  Settings → DNS Filtering → DNS Server → Add custom server
  Protocol: DNS-over-QUIC
  Server address: quic://dns.mullvad.net
```

For Android 9+ native Private DNS (DoT only, not DoQ):
```
Settings → Network & internet → Private DNS → Private DNS provider hostname
Enter: dns.mullvad.net
```

## Router-Level DoQ (OpenWrt)

OpenWrt with `odhcp6c` and `knot-resolver` supports DoQ:

```bash
# Install knot-resolver on OpenWrt
opkg install knot-resolver knot-resolver-mod-dns64

# /etc/knot-resolver/kresd.conf
-- Use DoQ upstream
policy.add(policy.all(policy.TLS_FORWARD({
  {'193.138.218.74', hostname='dns.mullvad.net'},
})))

-- Local cache
cache.size = 10 * MB

-- Listen on all interfaces for local clients
net.listen('0.0.0.0', 53, { kind = 'dns' })
```

## Verifying Encrypted DNS

```bash
# Confirm DNS queries are not going out in plaintext
# Run tcpdump while making a DNS request
sudo tcpdump -i eth0 -n port 53 &
curl https://example.com
# If you see UDP port 53 traffic → plaintext DNS is leaking

# If using DoQ correctly: you should see UDP port 784 (DoQ) or 443 (DoH)
sudo tcpdump -i eth0 -n udp port 784 &
q example.com @quic://dns.mullvad.net
# Should see traffic on 784

# Online leak test
curl https://browserleaks.com/dns  # requires browser
# Or:
curl https://api.ipify.org   # then check what resolvers your ISP sees
```

## Selecting a DoQ Provider by Privacy Policy

The strongest privacy option is a DoQ resolver with:
- No query logging
- Jurisdiction outside 14-Eyes countries
- Published technical audit

```
Mullvad (Sweden): audited, no logs, strong legal framework
AdGuard (Cyprus/international): no logs, open source clients
NextDNS (USA): logs optional and configurable per account
Quad9 (Switzerland): no logs, malware filtering, nonprofit
```

## Related Reading

- [How to Configure DNS over HTTPS for Privacy](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/)
- [Privacy-Focused DNS Filtering with AdGuard Home](/privacy-tools-guide/adguard-home-dns-filtering-setup/)
- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-tools-guide/privacy-dns-resolver-comparison-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
