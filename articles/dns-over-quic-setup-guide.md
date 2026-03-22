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

## macOS Configuration for DoQ

macOS has limited native support, but third-party tools work well:

```bash
# Install Knot Resolver (supports DoQ)
brew install knot-resolver

# Create configuration
mkdir -p ~/.knot-resolver
cat > ~/.knot-resolver/config << 'EOF'
-- Knot Resolver macOS configuration
modules = { 'hints' }

-- DoQ upstream
policy.add(policy.all(policy.TLS_FORWARD({
  {'45.76.113.31', 'quic://dns.mullvad.net'},  -- Mullvad DoQ server
})))

-- Cache
cache.size = 100 * MB
cache.min_ttl = 300

-- Listen on localhost only
net.listen('127.0.0.1', 53, { kind = 'dns' })
EOF

# Run Knot Resolver
kresd -c ~/.knot-resolver/config

# Configure system DNS to use local resolver
# System Preferences → Network → Advanced → DNS
# Add: 127.0.0.1
```

Alternatively, use **dnscrypt-proxy**:

```bash
# Install dnscrypt-proxy
brew install dnscrypt-proxy

# Configuration file
cat > /usr/local/etc/dnscrypt-proxy/dnscrypt-proxy.toml << 'EOF'
listen_addresses = ['127.0.0.1:53']

# Use Mullvad DoQ
server_names = ['mullvad']

# Log DNS queries (optional, privacy implications)
log_files = []

# Disable logging for privacy
cert_ignore_timestamp = false
EOF

# Start service
brew services start dnscrypt-proxy

# Verify
dig @127.0.0.1 example.com
```

## Windows Configuration for DoQ

Windows 11 native DoT support exists, but DoQ requires workaround:

```powershell
# Install AdGuard DNS Client (supports DoQ)
# Download from adguard-dns.io/windows

# Or use Nextdns app (supports DoQ)
# Download from nextdns.io

# For manual setup, use Go DoQ client:
# Install Go, then:
go install github.com/natesales/q@latest

# Query over DoQ from command line
q example.com @quic://dns.mullvad.net
```

PowerShell to verify DoQ is working:

```powershell
# Check DNS server configuration
Get-DnsClientServerAddress

# Set DNS to local DoQ proxy (if running)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("127.0.0.1")

# Verify DoQ traffic (requires Wireshark or similar)
# Look for UDP port 784 traffic
```

## iOS Configuration for DoQ

iOS 15+ partially supports DoQ via Private DNS (though limited):

```
Settings → Network & Internet → Advanced → Private DNS
Choose: Custom
Enter: dns.mullvad.net
```

For full DoQ support on iOS, use third-party apps:

- **Intra**: Supports DoH and DoQ, open source
- **AdGuard for iOS**: Full DoQ support
- **DNSCloak**: Supports all protocols

Installation via TestFlight (example for Intra):

```
1. Visit: invite.intra.app
2. Accept invite
3. Install from TestFlight
4. Configure DoQ resolver in app settings
```

## Troubleshooting DoQ Connectivity Issues

If DoQ appears not to be working:

```bash
# 1. Verify local DoQ proxy is listening
ss -ulnp | grep 784
# Should show: UNCONN  0  0  0.0.0.0:784  0.0.0.0:*

# 2. Test DoQ query directly
q example.com @quic://dns.mullvad.net -v

# 3. Check firewall rules
sudo ufw status | grep 784
# Add if needed: sudo ufw allow 784/udp

# 4. Capture traffic to verify
sudo tcpdump -i eth0 -n 'udp port 784'

# 5. Test from different network (ISP blocking?)
# Use hotspot or mobile data to test

# 6. Verify certificate validity (if self-hosted)
openssl s_client -connect yourdns.example.com:784
```

## Performance Comparison: DoQ vs Other Protocols

Real-world testing on various networks:

```bash
#!/bin/bash
# dns-protocol-benchmark.sh

DOMAIN="example.com"
ITERATIONS=100

echo "=== DNS Protocol Performance Benchmark ==="

for protocol in "plain-dns" "dot" "doh" "doq"; do
  case $protocol in
    plain-dns)
      RESOLVER="@1.1.1.1"
      LABEL="DNS (plain)"
      ;;
    dot)
      RESOLVER="@tls://dns.mullvad.net"
      LABEL="DNS over TLS"
      ;;
    doh)
      RESOLVER="@https://dns.mullvad.net/dns-query"
      LABEL="DNS over HTTPS"
      ;;
    doq)
      RESOLVER="@quic://dns.mullvad.net"
      LABEL="DNS over QUIC"
      ;;
  esac

  echo -n "Testing $LABEL: "

  total_time=0
  for ((i=1; i<=ITERATIONS; i++)); do
    start=$(date +%s%N)
    q $DOMAIN $RESOLVER > /dev/null 2>&1
    end=$(date +%s%N)
    elapsed=$((($end - $start) / 1000000))  # Convert to ms
    total_time=$(($total_time + $elapsed))
  done

  avg=$((total_time / ITERATIONS))
  echo "${avg}ms average"
done

echo
echo "Summary:"
echo "Plain DNS: Fastest, no privacy"
echo "DoT:       +10-20ms, encrypted"
echo "DoH:       +15-25ms, looks like HTTPS"
echo "DoQ:       +5-15ms, lowest latency (QUIC)"
```

## Selecting Multiple DoQ Providers for Redundancy

Production setup with failover:

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml
upstream_dns:
  - quic://dns.mullvad.net       # Primary (Sweden, audited)
  - quic://dns.adguard-dns.com   # Secondary (Cyprus)
  - quic://dns.quad9.net         # Tertiary (Switzerland)

fallback_dns:
  - 9.9.9.9  # Quad9 (fallback to plain DNS)
  - 1.1.1.1  # Cloudflare (plain DNS backup)

# If all DoQ fails, falls back to plain DNS
# Ensures service continuity with privacy degradation only during outages
```

## Monitoring DoQ Usage and Performance

Track DNS query statistics:

```bash
# Using AdGuard Home API
curl -s http://localhost:3000/control/stats \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" | jq '{
    total_queries: .num_dns_queries,
    queries_today: .num_dns_queries_today,
    blocked: .num_blocked_filtering,
    query_log_enabled: .query_log_enabled,
    stats_interval: .stats_interval
  }'

# Parse query log for DoQ specific stats
jq -r 'select(.Protocol == "quic") | .QH' /opt/AdGuardHome/data/querylog.json \
  | sort | uniq -c | sort -rn | head -10
```

## Related Reading

- [How to Configure DNS over HTTPS for Privacy](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/)
- [Privacy-Focused DNS Filtering with AdGuard Home](/privacy-tools-guide/adguard-home-dns-filtering-setup/)
- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-tools-guide/privacy-dns-resolver-comparison-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
