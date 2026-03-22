---
layout: default
title: "DNS over TLS Setup on Linux"
description: "Configure DNS over TLS (DoT) on Linux with systemd-resolved and on Android's Private DNS feature to encrypt DNS queries and prevent ISP snooping"
date: 2026-03-21
author: theluckystrike
permalink: /dns-over-tls-setup-linux-android/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---
=======
---
>>>>>>> Stashed changes

{% raw %}

Every domain name lookup you make is normally sent as plaintext UDP to port 53. Your ISP, your router, anyone on the network, and any government with access to network infrastructure can read exactly which websites you're looking up. DNS over TLS (DoT) encrypts these queries over a TCP connection to port 853, making them unreadable to network observers.

DoT is simpler than DNS over HTTPS (DoH) for system-level deployment: one dedicated port, standard TLS, no HTTP overhead. This guide covers setting it up on Linux using systemd-resolved and on Android using the built-in Private DNS feature.

## Key Takeaways

- **If you're in a**: restrictive environment where port 853 is blocked, DoH is the better choice.
- **For most privacy use**: cases (ISP snooping prevention, protecting queries on public Wi-Fi), DoT is simpler and sufficient.
- **If you use Mullvad VPN**: their resolver is the most privacy-preserving option since it operates entirely within their no-log infrastructure.
- **Use VPN DNS from**: provider that supports DoT # 2.
- **Cloudflare offers better performance**: in many regions but retains some logs for up to 24 hours.
- **Use these steps to confirm**: 1.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced DoT Configuration for Maximum Privacy](#advanced-dot-configuration-for-maximum-privacy)
- [Performance Monitoring](#performance-monitoring)
- [Troubleshooting DoT Issues](#troubleshooting-dot-issues)
- [Related Reading](#related-reading)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: How DoT Compares to DoH

Both protocols encrypt DNS, but they differ in deployment:

| Property | DNS over TLS (DoT) | DNS over HTTPS (DoH) |
|----------|--------------------|----------------------|
| Port | 853 (dedicated) | 443 (shared with HTTPS) |
| Visibility | Network operators can see DoT traffic | Blends in with HTTPS |
| Use case | System-level DNS | Browser-level, censorship circumvention |
| Blocking | Easy to block by port | Hard to block without breaking HTTPS |

DoT on port 853 is easier for network administrators to identify and optionally block. If you're in a restrictive environment where port 853 is blocked, DoH is the better choice. For most privacy use cases (ISP snooping prevention, protecting queries on public Wi-Fi), DoT is simpler and sufficient.

### Step 2: Choose a DoT Resolver

Your encrypted queries still end up at a resolver that can see them. Choose carefully:

| Resolver | DoT hostname | Privacy policy | Logging |
|----------|--------------|----------------|---------|
| Quad9 | `dns.quad9.net` | No personal data logged | Minimal |
| Cloudflare | `1dot1dot1dot1.cloudflare-dns.com` | 24h logging, then deleted | Yes, limited |
| Mullvad | `base.dns.mullvad.net` | No logging, requires Mullvad VPN | None |
| NextDNS | `[id].dns.nextdns.io` | Configurable, logs optional | Optional |
| dns.sb | `dot.sb` | No logging | None |

For most users, Quad9 provides a good balance: no logging, malware blocking, and operated by a non-profit. Cloudflare offers better performance in many regions but retains some logs for up to 24 hours. If you use Mullvad VPN, their resolver is the most privacy-preserving option since it operates entirely within their no-log infrastructure.

### Step 3: Linux: systemd-resolved Configuration

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

### Handling the DNSOverTLS=opportunistic vs. yes difference

The `DNSOverTLS` setting accepts three values with meaningfully different behavior:

- `no`: No encryption, plaintext DNS only
- `opportunistic`: Attempt DoT but fall back to plaintext if unavailable
- `yes`: Require DoT, fail if TLS cannot be established

For privacy, always use `yes`. The `opportunistic` mode silently falls back to unencrypted DNS when port 853 is blocked or the resolver doesn't support TLS, giving a false sense of security. Using `yes` ensures that if DoT fails, DNS queries fail entirely rather than leaking in plaintext.

```bash
# Verify your setting is enforced
sudo grep DNSOverTLS /etc/systemd/resolved.conf
# Should output: DNSOverTLS=yes
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

Stubby's key advantage over systemd-resolved's built-in DoT is support for public key pinning. The `tls_pubkey_pinset` directive pins the expected certificate fingerprint for each resolver, preventing MITM attacks even if a fraudulent certificate is issued. This matters in corporate or state-level network environments where TLS interception is practiced.

### Fixing Split DNS and VPN Conflicts

When you run a VPN, DNS traffic may route through the VPN's DNS servers rather than your configured DoT resolver. Check the effective resolver:

```bash
# While VPN is active
resolvectl status
# Look for the interface used by your VPN (e.g., tun0, wg0)
# and what DNS server is configured for that interface
```

To force all interfaces to use your DoT resolver regardless of VPN, add the per-link configuration:

```bash
# For WireGuard interfaces
sudo resolvectl dns wg0 9.9.9.9
sudo resolvectl dnsovertls wg0 yes
```

### Step 4: Android: Built-In Private DNS

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

### Verifying Android Private DNS is actually working

The Private DNS setting is easy to configure but easy to get wrong. Many users set it and assume it is working without verifying. Use these steps to confirm:

1. Open Quad9's test page: https://on.quad9.net — if DoT is working through Quad9, this page confirms it
2. Visit dnsleaktest.com → "Extended Test" — the responding server should be your chosen resolver
3. Run a packet capture using Android's built-in VPN feature with a packet capture app (PCAPdroid is open source) and look for traffic on port 853

One common failure mode: some carrier APN configurations intercept DNS traffic at the network level. In these cases, Private DNS may appear to be set correctly but queries are intercepted before reaching port 853. If dnsleaktest.com consistently shows your carrier's DNS despite correct Private DNS configuration, you may need a VPN to bypass this interception.

### NextDNS for Android: Per-Device Filtering and Logging Control

NextDNS deserves special mention for Android users who want both DoT privacy and DNS-level content filtering. After creating a free NextDNS account:

1. Navigate to your NextDNS dashboard and copy your profile ID
2. Set Private DNS to `[your-id].dns.nextdns.io`
3. In the NextDNS dashboard, enable the blocklists you want (ads, malware, trackers)
4. Under "Privacy," disable logging if you don't want query logs stored

This gives you per-device DNS filtering without requiring a separate DNS server or VPN, all running through standard Android DoT.

### Step 5: Limitations of DoT

**DoT encrypts DNS queries but not the SNI (Server Name Indication)** in TLS connections. When you connect to an HTTPS site, the hostname is visible in the TLS ClientHello even if DNS is encrypted. Encrypted Client Hello (ECH) addresses this but is not yet universally deployed.

**DoT does not provide anonymity.** Your chosen DoT resolver still sees all your queries. DoT is ISP-privacy, not anonymity. For full DNS anonymity, route queries through Tor or use a DNS resolver accessible only through a VPN.

**DoT requires trusting your resolver.** Switching from your ISP's resolver to Cloudflare or Quad9 moves trust, not eliminates it. Choose a resolver with a clear, audited no-logging policy.

**Port 853 is blockable.** Unlike DoH which uses port 443, DoT's dedicated port can be blocked by network administrators or censorship systems. In environments where port 853 is filtered, DoT silently fails unless you use strict mode—another reason to always set `DNSOverTLS=yes` rather than `opportunistic`.
## Advanced DoT Configuration for Maximum Privacy

For power users wanting additional hardening beyond basic setup:

### Certificate Pinning for DoT

Instead of trusting the entire certificate authority ecosystem, pin specific resolver certificates:

```bash
# Extract certificate from DoT resolver and pin it
# 1. Get the certificate chain
openssl s_client -connect dns.quad9.net:853 -showcerts < /dev/null 2>/dev/null | \
  openssl x509 -outform PEM -out quad9.pem

# 2. Calculate certificate hash for pinning
openssl x509 -in quad9.pem -noout -pubkey | \
  openssl pkey -pubin -outform DER | \
  openssl dgst -sha256 -binary | \
  openssl enc -base64

# 3. Add to systemd-resolved configuration
# In /etc/systemd/resolved.conf:
# [Resolve]
# DNS=9.9.9.9#dns.quad9.net
# DNSOverTLS=yes
```

### Multi-Resolver Failover with Health Checks

Implement automatic failover if your primary resolver becomes unreachable:

```bash
#!/bin/bash
# dns-failover-monitor.sh

RESOLVERS=(
  "9.9.9.9#dns.quad9.net"
  "1.1.1.1#1dot1dot1dot1.cloudflare-dns.com"
  "89.163.128.29#base.dns.mullvad.net"
)

HEALTH_CHECK_HOST="example.com"
ALERT_EMAIL="admin@example.com"

check_resolver_health() {
  local resolver=$1
  local ip=$(echo $resolver | cut -d'#' -f1)

  # Test DoT connection with timeout
  if timeout 2 openssl s_client -connect "$ip:853" -quiet < /dev/null >/dev/null 2>&1; then
    return 0
  else
    return 1
  fi
}

for resolver in "${RESOLVERS[@]}"; do
  if ! check_resolver_health "$resolver"; then
    echo "Resolver $resolver failed health check"
    # Switch to next resolver dynamically
    # Implement via D-Bus call to systemd-resolved
  fi
done
```

### Tor Integration for DNS Anonymity

Combine DoT with Tor for maximum privacy:

```bash
# 1. Install Tor
sudo apt install tor

# 2. Configure Tor for DNS over DoT
# In /etc/tor/torrc:
# DnsPort 53

# 3. Point systemd-resolved to Tor's DNS port
sudo nano /etc/systemd/resolved.conf
# [Resolve]
# DNS=127.0.0.1
# DNSOverTLS=yes

# 4. All DNS queries now route through Tor exit nodes
# Privacy: Your resolver cannot identify you
# Tradeoff: Slower due to Tor overhead
```

## Performance Monitoring

DoT adds slight latency. Monitor impact on your system:

```bash
#!/bin/bash
# Compare DoT vs standard DNS performance

echo "=== DNS Query Performance Comparison ==="

# Baseline: Standard DNS
echo "Standard DNS (port 53):"
time dig @8.8.8.8 +noall +answer example.com

# Test: DoT performance
echo ""
echo "DoT (port 853):"
time dig @9.9.9.9 -p 853 +tls +noall +answer example.com

# Measure latency increase
STANDARD=$(dig @8.8.8.8 +stats example.com | grep "Query time")
DOT=$(dig @9.9.9.9 -p 853 +tls +stats example.com | grep "Query time")

echo ""
echo "Standard DNS latency: $STANDARD"
echo "DoT latency: $DOT"
echo ""
echo "Note: DoT adds ~10-50ms typically, offset by privacy gains"
```

## Troubleshooting DoT Issues

### Symptom: DNS Queries Still Unencrypted

Verify actual protocol usage:

```bash
# Monitor traffic in real-time
sudo tcpdump -i any -n 'dst port 853 or (dst port 53 and not localhost)' | grep -E 'DNS|53|853'

# If you see port 53 traffic, DoT is not active
# Check systemd-resolved status
systemctl status systemd-resolved

# Force DoT retry with verbose logging
SYSTEMD_LOG_LEVEL=debug resolvectl query example.com
```

### Symptom: Some Domains Won't Resolve

Certain malformed domains or rare cases fail on DoT-only:

```bash
# Test fallback mechanism
resolvectl status | grep -A 2 "DNSSEC"

# If DoT fails, verify FallbackDNS is configured
# In /etc/systemd/resolved.conf:
# FallbackDNS=9.9.9.9

# Test specific domain resolution
dig @9.9.9.9 +tls problematic-domain.example
dig @9.9.9.9 problematic-domain.example  # Compare without TLS
```

### Symptom: VPN Conflicts with DoT

Some VPNs override DNS settings or block port 853:

```bash
# Verify whether port 853 is blocked
nmap -p 853 dns.quad9.net

# If blocked, switch to DoH or use VPN DNS directly
# Check which DNS your VPN is providing
resolvectl status

# For better DoT + VPN compatibility:
# 1. Use VPN DNS from provider that supports DoT
# 2. Or configure DNS at VPN tunnel level
# 3. Or use separate network namespace for DoT
```

### Step 6: Monitor DNS Queries

Log which sites are being resolved for privacy audits:

```bash
# Enable query logging (systemd-resolved)
sudo systemctl set-property systemd-resolved.service "ExecStart=" ""
sudo systemctl edit systemd-resolved.service

# Add to [Service] section:
# ExecStart=
# ExecStart=/lib/systemd/systemd-resolved --log-target=journal --log-level=debug

sudo systemctl restart systemd-resolved

# View query logs
journalctl -u systemd-resolved -f | grep "query"

# Analysis: Extract queried domains
journalctl -u systemd-resolved | grep "query" | \
  sed 's/.*query //' | \
  sort | uniq -c | sort -rn | head -20
```

## Related Reading

- [Android Privacy Best Practices 2026](/android-privacy-best-practices-2026/)
- [VPN Kill Switch Configuration on Linux](/vpn-kill-switch-linux-iptables-setup/)
- [How to Set Up Encrypted DNS over HTTPS on All Devices](/how-to-set-up-encrypted-dns-over-https-doh-on-all-devices-guide/)
- [AI Tools for Automating Cloud Security Compliance Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-automating-cloud-security-compliance-scanning-i/)
- [AI Container Security Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-container-security-scanning/)

## Related Articles

- [Encrypted DNS over HTTPS on Linux](/privacy-tools-guide/encrypted-dns-over-https-linux-setup)
- [How to Configure DNS Over HTTPS (DoH) for Privacy in 2026](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/)
- [How To Tell If Your Dns Has Been Hijacked Symptoms](/privacy-tools-guide/how-to-tell-if-your-dns-has-been-hijacked-symptoms-check/)
- [How to Set Up Encrypted DNS on All Devices 2026](/privacy-tools-guide/how-to-set-up-encrypted-dns-on-all-devices-2026/)
- [Encrypted Dns Messaging Combination How To Layer Privacy Pro](/privacy-tools-guide/encrypted-dns-messaging-combination-how-to-layer-privacy-pro/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to on linux?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
