---
layout: default
title: "How to Configure Pi-hole with Unbound DNS"
description: "Set up Pi-hole with a local Unbound recursive resolver to block ads and trackers while eliminating reliance on upstream DNS providers"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-configure-pihole-with-unbound-dns/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Pi-hole blocks ads and trackers at the DNS level for your entire network, but by default it forwards your DNS queries to an upstream provider like Google or Cloudflare. Those providers can see every domain your network requests. Combining Pi-hole with Unbound eliminates that dependency — Unbound resolves DNS recursively by querying root servers directly, keeping your DNS data on your network.

## Prerequisites

- A Raspberry Pi (any model), a small VPS, or any Linux box
- Pi-hole installed (or install it as part of this guide)
- Debian/Ubuntu/Raspberry Pi OS

## Step 1: Install Pi-hole

```bash
# Pi-hole one-line installer (review at install.pi-hole.net)
curl -sSL https://install.pi-hole.net | bash

# During installation:
# - Interface: eth0 (or your primary interface)
# - Upstream DNS: select any (we will replace this with Unbound)
# - Install web interface: yes
# - Block IPv6: yes (recommended)

# Note the admin password shown at end of installation
```

After installation, Pi-hole is accessible at `http://pi.hole/admin` from your network.

## Step 2: Install and Configure Unbound

```bash
sudo apt install unbound

# Download the current root hints file (list of DNS root servers)
sudo curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
```

Create `/etc/unbound/unbound.conf.d/pi-hole.conf`:

```conf
server:
    # Verbosity (0-5; 0 = quiet)
    verbosity: 0

    interface: 127.0.0.1
    port: 5335

    # Restrict to local queries only
    access-control: 127.0.0.0/8 allow
    access-control: 0.0.0.0/0 refuse

    # Root hints file
    root-hints: /var/lib/unbound/root.hints

    # Trust DNSSEC
    auto-trust-anchor-file: /var/lib/unbound/root.key

    # Hardening
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-referral-path: yes
    use-caps-for-id: no

    # Privacy settings
    hide-identity: yes
    hide-version: yes
    qname-minimisation: yes       # Only send the minimum required name to each resolver
    qname-minimisation-strict: no # Fall back if strict mode causes failures

    # Caching
    cache-min-ttl: 3600
    cache-max-ttl: 86400

    # Performance
    prefetch: yes
    prefetch-key: yes
    num-threads: 1

    # Private addresses (never forward these to external resolvers)
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

## Step 3: Start Unbound and Verify

```bash
sudo systemctl enable --now unbound

# Test that Unbound resolves correctly
dig @127.0.0.1 -p 5335 google.com

# Check DNSSEC validation
dig @127.0.0.1 -p 5335 sigfail.verteiltesysteme.net | grep -i "SERVFAIL\|status"
# Should return SERVFAIL (DNSSEC validation caught a bad signature)

dig @127.0.0.1 -p 5335 sigok.verteiltesysteme.net | grep -i "NOERROR\|status"
# Should return NOERROR (valid signature)

# Check recursive resolution (no cached results)
dig @127.0.0.1 -p 5335 +trace example.com
```

## Step 4: Point Pi-hole to Unbound

In the Pi-hole admin panel:

1. Go to **Settings** > **DNS**
2. Uncheck all upstream DNS servers
3. In "Custom 1 (IPv4)", enter: `127.0.0.1#5335`
4. Check "Never forward non-FQDNs" and "Never forward reverse lookups"
5. Click **Save**

Or configure via the CLI:
```bash
pihole -a setdns "127.0.0.1#5335"
```

Verify Pi-hole is using Unbound:
```bash
# Query through Pi-hole
dig @127.0.0.1 example.com

# Check Pi-hole query log (should show queries forwarded to 127.0.0.1#5335)
pihole -t   # Tail the query log
```

## Step 5: Configure Your Network to Use Pi-hole

### Router-Level Configuration (Recommended)

Set your router's DHCP server to advertise Pi-hole's IP as the DNS server for all clients:

```
Router settings (varies by model):
DHCP → DNS Server 1: [Pi-hole's IP address, e.g., 192.168.1.10]
DNS Server 2: leave blank (forces all clients to use Pi-hole only)
```

### Individual Device Configuration

For devices not on your home network (laptops traveling):
```bash
# On Linux
sudo tee /etc/systemd/resolved.conf.d/pihole.conf > /dev/null <<'EOF'
[Resolve]
DNS=192.168.1.10
FallbackDNS=
EOF

sudo systemctl restart systemd-resolved
```

## Step 6: Add Privacy Blocklists

Pi-hole's default blocklist blocks ads. Add tracker-focused lists:

```bash
# Via admin panel: Group Management > Adlists > Add
# Recommended lists:
# https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
# https://blocklistproject.github.io/Lists/tracking.txt
# https://s3.amazonaws.com/lists.disconnect.me/simple_tracking.txt
# https://raw.githubusercontent.com/crazy-max/WindowsSpyBlocker/master/data/hosts/spy.txt

# Update gravity (apply new blocklists)
pihole -g
```

Check your block count:
```bash
pihole status
# Shows current blocklist size and number of queries blocked today
```

## Step 7: QNAME Minimisation Explained

The `qname-minimisation: yes` setting in Unbound is important for privacy. Without it, when resolving `mail.google.com`, Unbound sends the full name `mail.google.com` to each DNS server it queries (including root servers and TLD servers). With minimisation enabled:

- Root servers only see: `.com`
- TLD servers only see: `google.com`
- Google's authoritative server only sees: `mail.google.com`

Each server sees only what it needs to answer its portion of the query — root servers never learn what specific domains you visit.

## Monitoring and Maintenance

```bash
# View top blocked domains
pihole -c   # Query the FTL database

# Check daily query stats
pihole -c -e

# Update Pi-hole
pihole -up

# Update blocklists
pihole -g

# Check Unbound performance
unbound-control stats | grep -E "total\.|num\."
```

Keep root hints updated (root servers change occasionally):
```bash
# Add to cron
echo "0 3 1 * * root curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.root && systemctl restart unbound" | \
  sudo tee /etc/cron.d/update-root-hints
```

## Related Articles

- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-tools-guide/privacy-dns-resolver-comparison-2026/)
- [Encrypted DNS over HTTPS on Linux](/privacy-tools-guide/encrypted-dns-over-https-linux-setup)
- [Home Network Privacy Pihole Dns Filtering Guide 2026](/privacy-tools-guide/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [Configure Private DNS on Android for System-Wide Tracker](/privacy-tools-guide/how-to-configure-private-dns-on-android-for-system-wide-trac/)
- [Privacy-Focused DNS Providers Comparison 2026](/privacy-tools-guide/privacy-focused-dns-providers-comparison-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
