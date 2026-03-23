---
layout: default
title: "OpenWrt VPN Router Setup with WireGuard"
description: "How to configure OpenWrt as a WireGuard VPN router so that all devices on your home network route through the VPN without per-device configuration"
date: 2026-03-21
author: theluckystrike
permalink: /openwrt-vpn-router-wireguard-setup/
categories: [guides, security]
tags: [privacy-tools-guide, vpn]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Running a VPN on each individual device means 8 apps to configure, update, and maintain. A VPN at the router level means every device on your network. phone, laptop, smart TV, game console, IoT sensors. is protected automatically, without installing anything on them.

This guide configures OpenWrt as a WireGuard VPN client that routes all LAN traffic through a WireGuard server (either a commercial VPN provider or a self-hosted server).

Prerequisites

- A router running OpenWrt (19.07 or later; 21.02+ preferred)
- A WireGuard server to connect to (your own or a VPN provider that gives WireGuard config files)
- SSH access to your OpenWrt router

If your router doesn't run OpenWrt, check the OpenWrt Table of Hardware (openwrt.org/toh/start) to see if it's supported.

Step 1: Install WireGuard on OpenWrt

Connect to your router via SSH:

```bash
ssh root@192.168.1.1
```

Install the required packages:

```bash
opkg update
opkg install wireguard-tools kmod-wireguard luci-app-wireguard

Reboot to load the kernel module
reboot
```

After rebooting, verify WireGuard is available:

```bash
wg --version
```

Step 2: Get or Create WireGuard Configuration

You need a WireGuard configuration from your VPN provider or your own server.

From a commercial VPN provider

Most providers (Mullvad, ProtonVPN, IVPN, AzireVPN) have a portal where you can generate WireGuard config files. Download the config file. it looks like this:

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY_HERE
Address = 10.67.0.2/32
DNS = 10.64.0.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY_HERE
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = vpn.example.com:51820
PersistentKeepalive = 25
```

Self-hosted WireGuard server

If you're connecting to your own WireGuard server, generate keys on the router:

```bash
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
cat /etc/wireguard/private.key
cat /etc/wireguard/public.key
```

Add the router's public key to your server's WireGuard configuration as a new peer.

Step 3: Configure WireGuard Interface

Create a new WireGuard interface in OpenWrt's UCI configuration:

```bash
Add WireGuard interface
uci set network.wg0=interface
uci set network.wg0.proto=wireguard
uci set network.wg0.private_key='YOUR_CLIENT_PRIVATE_KEY'
uci set network.wg0.listen_port=51820
uci add_list network.wg0.addresses='10.67.0.2/32'

Add the VPN server as a peer
uci set network.wgpeer=wireguard_wg0
uci set network.wgpeer.public_key='SERVER_PUBLIC_KEY'
uci set network.wgpeer.endpoint_host='vpn.example.com'
uci set network.wgpeer.endpoint_port='51820'
uci set network.wgpeer.route_allowed_ips='1'
uci set network.wgpeer.persistent_keepalive='25'
uci add_list network.wgpeer.allowed_ips='0.0.0.0/0'
uci add_list network.wgpeer.allowed_ips='::/0'

Commit and apply
uci commit network
/etc/init.d/network restart
```

Verify the tunnel is up:

```bash
wg show
Should show the peer with a recent handshake and data transfer
```

Step 4: Configure Firewall Rules

Add the WireGuard interface to the VPN firewall zone and create routing rules:

```bash
Add wg0 to WAN zone (it carries outbound traffic)
uci add_list firewall.@zone[1].network='wg0'

Allow forwarding from LAN to WireGuard zone
uci set firewall.vpn_fwd=rule
uci set firewall.vpn_fwd.src='lan'
uci set firewall.vpn_fwd.dest='wan'
uci set firewall.vpn_fwd.target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart
```

Step 5: Configure Routing

Set the WireGuard interface as the default route for all traffic:

```bash
Create a routing table entry that routes all traffic through wg0
This uses OpenWrt's routing metrics to prefer the VPN tunnel

Via UCI (simpler)
uci set network.wg0.defaultroute='1'

Or via direct ip command for immediate testing
ip route add default dev wg0

uci commit network
/etc/init.d/network restart
```

Test from the router:

```bash
Check what IP address appears when routing through the VPN
curl https://api.ipify.org
Should show the VPN server's IP, not your real IP
```

Test from a device on your LAN:

```bash
curl https://api.ipify.org
Should also show the VPN IP
```

Step 6: Configure DNS for VPN-Routed Traffic

When all traffic routes through the VPN, DNS queries should also go through the VPN's DNS server to prevent DNS leaks.

```bash
Set the VPN provider's DNS server
(from the DNS field in your WireGuard config file)
uci set dhcp.@dnsmasq[0].server='10.64.0.1'

Prevent dnsmasq from using the router's upstream DNS directly
uci set dhcp.@dnsmasq[0].noresolv='1'

Optionally add a fallback
uci add_list dhcp.@dnsmasq[0].server='9.9.9.9'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

Verify DNS is going through the VPN:

```bash
From a LAN device
dig @8.8.8.8 +short myip.opendns.com
If DNS is leaking, this will show your real IP
Should show VPN IP or fail if your VPN blocks external DNS
```

Step 7: Implement a Kill Switch

If the WireGuard tunnel drops, traffic should be blocked (not fall back to your real IP):

```bash
Add firewall rule: block LAN → WAN traffic except through wg0
uci add firewall.rule
uci set firewall.@rule[-1].name='VPN-killswitch'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].target='REJECT'
uci set firewall.@rule[-1].enabled='1'

This rule applies to the physical WAN. wg0 is in WAN zone but the
physical interface traffic (except VPN endpoint) is now rejected.
Adjust zone configuration if your VPN uses a separate zone.

uci commit firewall
/etc/init.d/firewall restart
```

A more thorough implementation requires separating the physical WAN from the VPN tunnel into different firewall zones. This is covered in OpenWrt's documentation for policy-based routing.

Verify the Setup

Run these tests from a device on your network:

```bash
Your visible IP should be the VPN server
curl -s https://api.ipify.org

DNS should not leak your real resolver
Visit: https://dnsleaktest.com and run extended test

Ensure WebRTC doesn't leak local IP (browser test)
Visit: https://browserleaks.com/webrtc

Test that traffic stops if VPN drops
Temporarily down the wg0 interface on the router:
ssh root@192.168.1.1 "ip link set wg0 down"
curl -s --max-time 5 https://api.ipify.org
Should time out (kill switch working) or return your real IP (kill switch needs adjustment)
Bring it back up: ssh root@192.168.1.1 "ip link set wg0 up"
```

Related Articles

- [How to Set Up a VPN on Your Router](/vpn-on-router-setup-guide/)
- [How to Set Up VPN on Router Firmware: Complete Guide](/how-to-set-up-vpn-on-router-firmware-level-guide/)
- [How to Use WireGuard for Self-Hosted VPN in 2026](/articles/how-to-use-wireguard-for-self-hosted-vpn-2026/---)
- [Set Up a Personal VPN with WireGuard](/wireguard-personal-vpn-setup-guide)
- [How to Set Up WireGuard VPN on VPS 2026](/how-to-set-up-wireguard-vpn-on-vps-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

Frequently Asked Questions

How long does it take to with wireguard?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
