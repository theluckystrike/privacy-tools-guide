---
layout: default
title: "DNS over TLS Setup on Linux and Android"
description: "Configure DNS over TLS (DoT) on Linux with systemd-resolved and on Android's Private DNS feature to encrypt DNS queries and prevent ISP snooping"
date: 2026-03-21
author: theluckystrike
permalink: /dns-over-tls-setup-linux-android/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Every domain name lookup you make is normally sent as plaintext UDP to port 53. Your ISP, your router, anyone on the network, and any government with access to network infrastructure can read exactly which websites you're looking up. DNS over TLS (DoT) encrypts these queries over a TCP connection to port 853, making them unreadable to network observers.

DoT is simpler than DNS over HTTPS (DoH) for system-level deployment: one dedicated port, standard TLS, no HTTP overhead. This guide covers setting it up on Linux using systemd-resolved and on Android using the built-in Private DNS feature.

## How DoT Compares to DoH

Both protocols encrypt DNS, but they differ in deployment:

| Property | DNS over TLS (DoT) | DNS over HTTPS (DoH) |
|----------|--------------------|----------------------|
| Port | 853 (dedicated) | 443 (shared with HTTPS) |
| Visibility | Network operators can see DoT traffic | Blends in with HTTPS |
| Use case | System-level DNS | Browser-level, censorship circumvention |
| Blocking | Easy to block by port | Hard to block without breaking HTTPS |

DoT on port 853 is easier for network administrators to identify and optionally block. If you're in a restrictive environment where port 853 is blocked, DoH is the better choice. For most privacy use cases (ISP snooping prevention, protecting queries on public Wi-Fi), DoT is simpler and sufficient.

## Choosing a DoT Resolver

Your encrypted queries still end up at a resolver that can see them. Choose carefully:

| Resolver | DoT hostname | Privacy policy | Logging |
|----------|--------------|----------------|---------|
| Quad9 | `dns.quad9.net` | No personal data logged | Minimal |
| Cloudflare | `1dot1dot1dot1.cloudflare-dns.com` | 24h logging, then deleted | Yes, limited |
| Mullvad | `base.dns.mullvad.net` | No logging, requires Mullvad VPN | None |
| NextDNS | `[id].dns.nextdns.io` | Configurable, logs optional | Optional |
| dns.sb | `dot.sb` | No logging | None |

For most users, Quad9 provides a good balance: no logging, malware blocking, and operated by a non-profit.

## Linux: systemd-resolved Configuration

`systemd-resolved` is the default DNS resolver on Ubuntu 20.04+, Fedora, Arch, and most modern Linux distributions.

### Check if systemd-resolved is active

```bash
systemctl status systemd-resolved
resolvectl status
```

### Configure DoT in systemd-resolved

Edit the main configuration:

```bash
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
# Use Quad9 as primary, Cloudflare as fallback
DNS=9.9.9.9 149.112.112.112 1.1.1.1 1.0.0.1
# DoT hostnames for certificate validation
DNSOverTLS=yes
# Optional: also set fallback in case DoT fails
FallbackDNS=9.9.9.9 149.112.112.112
# Enable DNSSEC validation
DNSSEC=yes
# Enable DNS caching
Cache=yes
```

Restart the resolver:

```bash
sudo systemctl restart systemd-resolved
```

### Verify DoT is working

```bash
# Check resolver status — should show DNS-over-TLS and server names
resolvectl status

# Query a domain and check which server answered
resolvectl query example.com

# Use dig to test the DoT resolver directly
dig @9.9.9.9 -p 853 +tls example.com

# Check for DNS leaks
# Visit browserleaks.com/dns or dnsleaktest.com and run the test
# Should show only your chosen resolver, not your ISP's DNS
```

### Using Stubby for More Control

`stubby` is a DoT-specific resolver stub that provides finer-grained configuration:

```bash
sudo apt install stubby

sudo nano /etc/stubby/stubby.yml
```

```yaml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
appdata_dir: "/var/cache/stubby"

listen_addresses:
  - 127.0.0.1@53
  - 0::1@53

upstream_recursive_servers:
  - address_data: 9.9.9.9
    tls_auth_name: "dns.quad9.net"
    tls_pubkey_pinset:
      - digest: "sha256/pin_value_here"
  - address_data: 149.112.112.112
    tls_auth_name: "dns.quad9.net"
  - address_data: 1.1.1.1
    tls_auth_name: "1dot1dot1dot1.cloudflare-dns.com"
```

```bash
sudo systemctl enable stubby
sudo systemctl start stubby

# Point resolved to stubby
sudo nano /etc/systemd/resolved.conf
# DNS=127.0.0.1
# DNSOverTLS=no   (stubby handles TLS before resolved sees it)
```

## Android: Built-In Private DNS

Android 9 (Pie) and later include native DoT support called "Private DNS."

Settings → Network & internet → Advanced → Private DNS:
- Select "Private DNS provider hostname"
- Enter the hostname of your DoT resolver

Quad9: `dns.quad9.net`
Cloudflare: `1dot1dot1dot1.cloudflare-dns.com`
NextDNS (use your account ID): `[id].dns.nextdns.io`

Android will attempt to connect to port 853 on this hostname using TLS. If the connection fails, it falls back to your regular DNS unless you set "Private DNS" to strict mode via ADB:

```bash
# Force strict mode (no fallback if DoT fails — may break internet access)
adb shell settings put global private_dns_mode hostname
adb shell settings put global private_dns_specifier dns.quad9.net
```

### Verify Android DoT is working

Use the DNS leak test:
1. Open Quad9's test page: https://on.quad9.net
2. Or visit dnsleaktest.com → "Extended Test"
3. Check that the responding DNS server is your chosen resolver, not your ISP's

## Limitations of DoT

**DoT encrypts DNS queries but not the SNI (Server Name Indication)** in TLS connections. When you connect to an HTTPS site, the hostname is visible in the TLS ClientHello even if DNS is encrypted. Encrypted Client Hello (ECH) addresses this but is not yet universally deployed.

**DoT does not provide anonymity.** Your chosen DoT resolver still sees all your queries. DoT is ISP-privacy, not anonymity. For full DNS anonymity, route queries through Tor or use a DNS resolver accessible only through a VPN.

**DoT requires trusting your resolver.** Switching from your ISP's resolver to Cloudflare or Quad9 moves trust, not eliminates it. Choose a resolver with a clear, audited no-logging policy.

## Related Reading

- [Android Privacy Best Practices 2026](/android-privacy-best-practices-2026/)
- [VPN Kill Switch Configuration on Linux](/vpn-kill-switch-linux-iptables-setup/)
- [How to Set Up Encrypted DNS over HTTPS on All Devices](/how-to-set-up-encrypted-dns-over-https-doh-on-all-devices-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
