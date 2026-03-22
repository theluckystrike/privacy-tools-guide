---
layout: default
title: "How To Configure Openwrt Guest Network With Separate Dns"
description: "A practical guide for developers and power users to set up OpenWRT guest networks with custom DNS servers and strict firewall isolation"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-openwrt-guest-network-with-separate-dns-and/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

OpenWRT provides powerful network isolation capabilities that go beyond simple WiFi separation. By configuring a guest network with separate DNS resolution and firewall rules, you can create air-gapped network segments that prevent guest devices from accessing your main network while forcing DNS queries through trusted servers. This guide walks through the complete configuration using both the LuCI web interface and command-line tools.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Create the Guest Network Interface](#step-1-create-the-guest-network-interface)
- [Step 2: Configure Separate DNS Resolution](#step-2-configure-separate-dns-resolution)
- [Step 3: Set Up Wireless for Guest Access](#step-3-set-up-wireless-for-guest-access)
- [Step 4: Configure Firewall Isolation Rules](#step-4-configure-firewall-isolation-rules)
- [Step 5: Apply and Verify Configuration](#step-5-apply-and-verify-configuration)
- [Testing the Setup](#testing-the-setup)
- [Advanced: VLAN-Based Isolation](#advanced-vlan-based-isolation)
- [Preventing DNS Leaks on the Guest Network](#preventing-dns-leaks-on-the-guest-network)
- [Captive Portal for Guest Authentication](#captive-portal-for-guest-authentication)
- [Rate Limiting and Bandwidth Fairness](#rate-limiting-and-bandwidth-fairness)
- [Monitoring Guest Activity](#monitoring-guest-activity)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before starting, ensure you have:
- OpenWRT 21.02 or later installed on your router
- SSH access to the router
- A secondary WiFi interface or VLAN-capable hardware

## Step 1: Create the Guest Network Interface

Using the command line, create a new interface for guest devices. Edit `/etc/config/network`:

```bash
config interface 'guest'
    option device 'br-guest'
    option proto 'static'
    option ipaddr '10.0.2.1'
    option netmask '255.255.255.0'
    option ip6assign '64'
```

This configuration creates a dedicated subnet (10.0.2.0/24) for guest devices. The router itself will use 10.0.2.1 as its gateway address.

Next, add a DHCP server for this interface:

```bash
config dhcp 'guest'
    option interface 'guest'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dns '1.1.1.1 8.8.8.8'
```

This allocates IPs 10.0.2.100-10.0.2.249 to guests and assigns Cloudflare and Google DNS to connected devices automatically.

## Step 2: Configure Separate DNS Resolution

To force guest traffic through specific DNS servers rather than your router's default resolver, install and configure `dnsmasq` properly. Edit `/etc/config/dhcp`:

```bash
config dnsmasq 'guest_dns'
    option interface 'guest'
    option rebind_protection '1'
    option domainneeded '1'
    option localise_queries '1'
    option cachesize '1000'
    list server '1.1.1.1'
    list server '2606:4700:4700::1111'
    list domain 'guest.local'
```

This creates an isolated DNS instance that only serves the guest network. The `rebind_protection` and `domainneeded` options prevent DNS rebinding attacks. Guests cannot see your internal network domain names.

For encrypted DNS (DoH/DoT), consider installing `dnscrypt-proxy` and forwarding queries from the guest interface:

```bash
config dnsmasq
    option noresolv '1'
    list server '127.0.0.1#5353'
```

This sends all DNS queries through the local DoH proxy running on port 5353.

## Step 3: Set Up Wireless for Guest Access

Create a dedicated WiFi network for guests in `/etc/config/wireless`:

```bash
config wifi-iface 'guest_wifi'
    option device 'radio0'
    option mode 'ap'
    option ssid 'Guest-Network'
    option encryption 'psk2'
    option key 'strong-password-here'
    option network 'guest'
    option isolate '1'
```

The `isolate '1'` option is critical—it prevents guest devices from communicating directly with each other, adding another layer of security.

## Step 4: Configure Firewall Isolation Rules

The firewall is where true network separation happens. Edit `/etc/config/firewall` to create strict rules:

### Deny Guest to Main Network

```bash
config zone
    option name 'guest'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    list network 'guest'
```

This zone configuration rejects all incoming connections and forwards from the guest network. Guests can initiate outbound connections but cannot access internal services.

### Allow Guest to WAN

```bash
config forwarding
    option src 'guest'
    option dest 'wan'
```

This allows guests to access the internet through your WAN connection while blocking access to the LAN.

### Block Guest to LAN

```bash
config rule
    option name 'Block-Guest-to-LAN'
    option src 'guest'
    option dest 'lan'
    option proto 'all'
    option target 'REJECT'
```

Explicitly block any traffic attempting to traverse from guest to LAN zones.

### Allow Guest DNS Queries

```bash
config rule
    option name 'Allow-Guest-DNS'
    option src 'guest'
    option dest_port '53'
    option proto 'tcpudp'
    option target 'ACCEPT'
```

This permits DNS queries through the firewall to your configured DNS servers.

## Step 5: Apply and Verify Configuration

Apply all changes and verify the setup:

```bash
# Restart services
/etc/init.d/network restart
/etc/init.d/firewall restart
/etc/init.d/dnsmasq restart

# Verify interfaces
ip addr show br-guest

# Check firewall rules
iptables -L guest_forward -v -n
```

You should see the guest bridge interface with your assigned subnet and firewall chains properly configured.

## Testing the Setup

Connect a device to the guest WiFi and verify:

1. **IP Assignment**: Confirm the device receives an IP in the 10.0.2.x range
2. **DNS Resolution**: Test with `nslookup example.com` — should resolve through your configured servers
3. **Isolation Test**: Attempt to ping your main router (e.g., 192.168.1.1) — should fail
4. **Internet Access**: Confirm web browsing works through the guest network

## Advanced: VLAN-Based Isolation

For hardware with limited radio interfaces, use VLANs to create multiple guest networks:

```bash
config interface 'guest_vlan'
    option device 'br-guest.10'
    option proto 'static'
    option ipaddr '10.0.3.1'
    option netmask '255.255.255.0'
```

Tag VLAN 10 on your switch ports and configure separate DHCP pools for each virtual guest network.

## Preventing DNS Leaks on the Guest Network

One common gap in guest network configurations is DNS leak prevention. Even after assigning custom DNS servers through DHCP, a determined guest device may attempt to use hardcoded DNS servers, bypassing your configuration entirely.

To intercept and redirect all DNS traffic on the guest interface, add a NAT rule that forces all DNS queries to your router's dnsmasq instance:

```bash
config rule
    option name 'Redirect-Guest-DNS'
    option src 'guest'
    option dest_port '53'
    option proto 'tcpudp'
    option target 'DNAT'
    option dest_ip '10.0.2.1'
```

This redirects any DNS query destined for an external server to your local resolver. Combined with a blocklist-enabled dnsmasq configuration, this approach stops tracking domains at the DNS layer for all guest devices, regardless of their individual DNS settings.

Additionally, consider blocking DNS-over-HTTPS at the firewall level. Applications like Chrome and Firefox will attempt DoH using well-known IP addresses. You can block the most common DoH providers:

```bash
# Block Cloudflare DoH
config rule
    option name 'Block-Guest-DoH-Cloudflare'
    option src 'guest'
    option dest 'wan'
    option dest_ip '1.1.1.1'
    option dest_port '443'
    option proto 'tcp'
    option target 'REJECT'
```

This is an advanced mitigation. For most home use cases, redirecting port 53 is sufficient.

## Captive Portal for Guest Authentication

In shared living situations, small offices, or AirBnB setups, you may want guests to acknowledge terms of service before accessing the internet. OpenWRT supports this through the `nodogsplash` captive portal package:

```bash
opkg install nodogsplash

# Basic nodogsplash configuration at /etc/config/nodogsplash
config nodogsplash
    option enabled '1'
    option gatewayinterface 'br-guest'
    option maxclients '20'
    option sessiontimeout '86400'
    option preauthidletimeout '30'
    option authauthidletimeout '3600'
```

When a guest connects and opens a browser, nodogsplash intercepts the request and redirects to your splash page. After the guest clicks through, the portal logs their MAC address and session start time and grants access. This provides a lightweight accountability layer without requiring login credentials.

## Rate Limiting and Bandwidth Fairness

Guest networks benefit from bandwidth limits so a single guest cannot saturate your connection. OpenWRT's traffic shaping supports per-interface limits through the `qosify` or `tc` utilities:

```bash
# Install simple bandwidth limiter
opkg install luci-app-sqm

# Set download/upload limits for guest interface
uci set sqm.guest=queue
uci set sqm.guest.interface='br-guest'
uci set sqm.guest.download='20000'
uci set sqm.guest.upload='10000'
uci commit sqm
/etc/init.d/sqm restart
```

This example limits guest traffic to 20 Mbps down and 10 Mbps up. Adjust values based on your connection capacity and how many guests you expect.

## Monitoring Guest Activity

Maintaining visibility into what your guest network is doing is good security hygiene. The `ntopng` package provides a lightweight traffic analysis dashboard:

```bash
opkg install ntopng
```

After installation, configure it to monitor the `br-guest` interface specifically. You can see active connections, bandwidth usage per client, and flagged suspicious domains in real time. This is particularly useful if you run a guest network for a small business or AirBnB situation where you want to ensure the network is not being misused.

For simpler logging, the standard dnsmasq query log captures all DNS lookups on the guest interface:

```bash
# /etc/config/dhcp - enable query logging
config dnsmasq
    option logqueries '1'
    option logfacility '/var/log/dnsmasq-guest.log'
```

Review this log periodically to identify unusual lookup patterns that might indicate a compromised guest device.

## Troubleshooting

Common issues and solutions:

- **Guest devices can't get IP**: Verify the DHCP server is enabled and the interface is in the correct firewall zone
- **DNS not working**: Ensure the dnsmasq instance is listening on the guest interface and firewall allows port 53
- **Guest can access LAN**: Check that the `block-guest-to-lan` firewall rule exists and is before any forwarding rules
- **Guest devices see each other**: Confirm `option isolate '1'` is set in the wireless configuration and restart the wireless subsystem
- **Slow DNS on guest**: If using dnscrypt-proxy or DoH forwarding, verify the upstream resolver is responding; fall back to direct DNS temporarily to isolate the bottleneck
- **VLAN traffic not tagging correctly**: Use `swconfig show` to inspect your switch port configuration and confirm the VLAN is properly tagged on the uplink port

## Frequently Asked Questions

**How long does it take to configure openwrt guest network with separate dns?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Create Separate Network Segment for Smart Home Isolating](/privacy-tools-guide/how-to-create-separate-network-segment-for-smart-home-isolat/)
- [How to Configure DNS over HTTPS Inside a VPN Tunnel](/privacy-tools-guide/how-to-configure-dns-over-https-inside-vpn-tunnel/)
- [Configure Private DNS on Android for System-Wide Tracker](/privacy-tools-guide/how-to-configure-private-dns-on-android-for-system-wide-trac/)
- [How to Configure VPN Exempt List for Local Network Access](/privacy-tools-guide/how-to-configure-vpn-exempt-list-for-local-network-access/)
- [Home Network Privacy Pihole Dns Filtering Guide 2026](/privacy-tools-guide/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [AI Coding Assistant for Network Traffic Analysis: What](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-network-traffic-analysis-what-connection/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
