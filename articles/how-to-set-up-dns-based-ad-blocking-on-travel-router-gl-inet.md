---

layout: default
title: "How to Set Up DNS-Based Ad Blocking on Travel Router GL-Inet for Privacy"
description: "A practical guide to configuring DNS-based ad blocking on GL-Inet travel routers. Protect your devices network-wide with AdGuard Home or Pi-hole deployment."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-dns-based-ad-blocking-on-travel-router-gl-inet/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

DNS-based ad blocking intercepts domain name resolution requests and blocks queries matching known advertising and tracking domains. When configured on your router, this approach provides network-wide protection without installing software on individual devices. GL-Inet travel routers running OpenWrt-based firmware support several DNS blocking solutions, with AdGuard Home being the most straightforward option for most users.

This guide covers deploying AdGuard Home directly on GL-Inet routers and running a separate Pi-hole instance for users who prefer dedicated hardware.

## Why DNS-Based Blocking Matters

Every device on your network makes DNS queries to resolve domain names into IP addresses. Advertisers and trackers host their content on separate domains, and blocking those domains at the DNS level prevents the requests from ever reaching your devices. This method is efficient because a single DNS block eliminates multiple tracking scripts, pixel tags, and advertisement resources simultaneously.

Travel routers present a unique use case. When connecting to hotel WiFi, public networks, or guest networks at Airbnb accommodations, your devices inherit whatever DNS configuration the network provides. Running your own DNS resolver on a travel router ensures consistent privacy protection regardless of the upstream network.

## Prerequisites

This guide assumes you have a GL-Inet travel router such as the GL-MT1300 (Beryl), GL-SFT1200 (Slate), or GL-AX1800 (Flint). These devices run OpenWrt-based firmware and have sufficient storage for running DNS blocking software.

Verify your router's available storage before proceeding:

```bash
df -h
```

AdGuard Home requires approximately 100MB of storage. If your device has limited space, consider using a USB drive for additional storage.

## Installing AdGuard Home on GL-Inet

GL-Inet routers running firmware version 3.x or later include application support through the web interface. The simplest installation method uses the built-in plugin system.

### Method 1: Web Interface Installation

1. Access your router's admin panel at `192.168.8.1` (default)
2. Navigate to **Applications** → **AdGuard Home**
3. Click **Install** and wait for the package to download
4. Configure the listening interface (typically `0.0.0.0` for all interfaces)
5. Set the upstream DNS servers (see configuration section below)

### Method 2: Command-Line Installation

For advanced users preferring terminal access:

```bash
# Update package lists
opkg update

# Install AdGuard Home
opkg install adguardhome

# Configure initialization
/etc/init.d/adguardhome enable
/etc/init.d/adguardhome start
```

After installation, access the AdGuard Home web interface at `192.168.8.1:3000` (or the port you configured during setup).

## Configuring Upstream DNS Servers

AdGuard Home requires upstream DNS resolvers to forward legitimate queries. For privacy-conscious users, consider these options:

### Cloudflare (Fast, No Logging)
```
1.1.1.1
1.0.0.1
```

### Quad9 (Security-Focused)
```
9.9.9.9
149.112.112.112
```

### NextDNS (Customizable)
```
45.90.28.0
45.90.30.0
```

To configure upstream DNS in AdGuard Home, navigate to **Settings** → **DNS Settings** and enter your chosen servers. For maximum privacy, use DNS-over-HTTPS by entering URLs such as:

```
https://dns.cloudflare.com/dns-query
https://dns.quad9.net/dns-query
```

## Configuring Client DNS

After setting up AdGuard Home, configure your router to use it for all DHCP-assigned DNS. In the GL-Inet web interface:

1. Go to **Network** → **DHCP and DNS**
2. In the **DNS Forwarding** section, enter `127.0.0.1#53`
3. Save and apply settings

This configuration ensures all devices connected to your router use the local DNS resolver for their queries.

## Managing Block Lists

AdGuard Home ships with default block lists, but you can add more comprehensive lists for better coverage. Navigate to **Filters** → **DNS Blocklists** and add these reputable sources:

### Recommended Block Lists

```
https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://v.firebog.net/hosts/Prigent-Ads.txt
```

Update your block lists regularly by clicking **Update** in the web interface or running:

```bash
adguardhome --update
```

## Setting Up Pi-hole on Separate Hardware

For users seeking more robust filtering or running additional network services, Pi-hole provides a mature alternative. Install Pi-hole on a Raspberry Pi or virtual machine, then configure your GL-Inet router to use it.

### Router Configuration

```bash
# SSH into your GL-Inet router
ssh root@192.168.8.1

# Navigate to network configuration
uci set dhcp.@dnsmasq[0].server='192.168.8.100#53'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

Replace `192.168.8.100` with your Pi-hole's IP address.

### Testing Your Configuration

Verify DNS blocking is working by querying a known advertising domain:

```bash
nslookup doubleclick.net 192.168.8.1
```

A blocked response shows the NXDOMAIN status. In AdGuard Home, check the **Query Log** to see blocked requests in real-time.

## Using Encrypted DNS

For complete privacy, configure your DNS resolver to use encrypted protocols. AdGuard Home supports DNS-over-HTTPS (DoH), DNS-over-TLS (DoT), and DNS-over-QUIC.

In AdGuard Home, navigate to **Settings** → **Encryption** and enable:

- **DNS-over-HTTPS** with certificate verification
- **DNS-over-TLS** with strict checking

This prevents upstream DNS providers from seeing your query history, complementing the blocking functionality.

## Troubleshooting Common Issues

### Devices Not Resolving Domains

If connected devices cannot resolve domains, check that the DNS forwarder is running:

```bash
/etc/init.d/adguardhome status
```

Restart if necessary:

```bash
/etc/init.d/adguardhome restart
```

### Slow DNS Resolution

Slow resolution typically indicates upstream DNS issues. Switch to faster servers like Cloudflare or add multiple upstream providers for redundancy.

### Block Lists Not Updating

Ensure your router has internet connectivity and can reach the block list URLs. Test with:

```bash
curl -I https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

## Practical Applications for Travel Routers

Deploying DNS blocking on a travel router provides consistent protection across all your devices. When staying at hotels or using public WiFi, connect your laptop, phone, and tablet to your personal travel router instead of the venue's network. Your DNS queries remain private, and advertising domains get blocked regardless of the upstream network's configuration.

This setup is particularly valuable for sensitive browsing or when working remotely. The router acts as a privacy firewall, filtering traffic at the DNS level before it reaches your devices.

## Related Reading

- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [How to Set Up VPN on Router Firmware: Complete Guide](/privacy-tools-guide/how-to-set-up-vpn-on-router-firmware-level-guide/)
- [How to Set Up Private DNS on Android for All Apps](/privacy-tools-guide/how-to-set-up-private-dns-on-android-for-all-apps/)
- [How to Set Up Encrypted DNS to Bypass DNS Poisoning in.](/privacy-tools-guide/how-to-set-up-encrypted-dns-to-bypass-dns-poisoning-in-censo/)

Built by