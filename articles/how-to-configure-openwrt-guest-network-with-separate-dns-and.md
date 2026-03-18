---
layout: default
title: "How to Configure OpenWRT Guest Network with Separate DNS and Firewall Isolation Rules"
description: "A practical guide for developers and power users to set up OpenWRT guest networks with custom DNS servers and strict firewall isolation."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-openwrt-guest-network-with-separate-dns-and/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

OpenWRT provides powerful network isolation capabilities that go beyond simple WiFi separation. By configuring a guest network with separate DNS resolution and firewall rules, you can create air-gapped network segments that prevent guest devices from accessing your main network while forcing DNS queries through trusted servers. This guide walks through the complete configuration using both the LuCI web interface and command-line tools.

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

## Troubleshooting

Common issues and solutions:

- **Guest devices can't get IP**: Verify the DHCP server is enabled and the interface is in the correct firewall zone
- **DNS not working**: Ensure the dnsmasq instance is listening on the guest interface and firewall allows port 53
- **Guest can access LAN**: Check that the `block-guest-to-lan` firewall rule exists and is before any forwarding rules

## Summary

A properly configured OpenWRT guest network with separate DNS and firewall isolation protects your main network from untrusted devices while giving guests internet access. The key components are: dedicated subnet routing, isolated DNS resolution, wireless client isolation, and explicit firewall deny rules between zones.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
