---
layout: default
title: "How to Set Up a VPN on Your Router"
description: "Configure WireGuard or OpenVPN on your router with OpenWrt or DD-WRT to protect every device on your network without per-device clients"
date: 2026-03-22
author: theluckystrike
permalink: /vpn-on-router-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up a VPN on Your Router

Running a VPN on each device individually means constant maintenance. A router-level VPN tunnels every device on your network — phones, smart TVs, consoles, IoT gadgets — through the VPN without installing anything on them.

This guide covers WireGuard setup on OpenWrt (the best modern option) and briefly covers OpenVPN for routers that don't support WireGuard.

---

## Router Requirements

Not every router can run a VPN — you need one with custom firmware support or a VPN-capable chipset.

**Supported options:**

| Option | Notes |
|--------|-------|
| OpenWrt-compatible routers | GL.iNet, Linksys WRT series, many TP-Link models |
| GL.iNet routers | WireGuard built into stock firmware |
| pfSense/OPNsense boxes | PC-based, most capable |
| Asus Merlin firmware | Supports OpenVPN and WireGuard on Asus routers |

Check `openwrt.org/toh` for hardware compatibility before buying.

**Minimum specs for WireGuard:**
- 128MB RAM
- 16MB flash
- MIPS or ARM CPU (ARM preferred for speed)

---

## Flash OpenWrt (if needed)

```bash
# Check your router model at https://openwrt.org/toh
# Download the correct factory image for your model

# Access router admin → Firmware upgrade
# Upload the OpenWrt factory image
# Wait 3-5 minutes — do not power off

# After reboot, SSH in:
ssh root@192.168.1.1
# Default: no password, set one immediately:
passwd
```

---

## Install WireGuard on OpenWrt

```bash
# SSH into your router
ssh root@192.168.1.1

# Update package list and install WireGuard
opkg update
opkg install wireguard-tools luci-app-wireguard kmod-wireguard

# Generate router's WireGuard keys
wg genkey | tee /etc/wireguard/router_private.key | wg pubkey > /etc/wireguard/router_public.key
chmod 600 /etc/wireguard/router_private.key

# Display keys
echo "Private: $(cat /etc/wireguard/router_private.key)"
echo "Public:  $(cat /etc/wireguard/router_public.key)"
```

---

## Configure WireGuard Client (Connecting to VPN Provider)

This example uses Mullvad, but works with any WireGuard VPN provider. Download your provider's WireGuard config file and extract the values.

```bash
# Create WireGuard interface config
cat > /etc/config/network_wg << 'EOF'
# Values from your VPN provider's WireGuard config

PRIVATE_KEY="your_private_key_here"
SERVER_PUBKEY="server_public_key_here"
SERVER_ENDPOINT="1.2.3.4:51820"
ALLOWED_IPS="0.0.0.0/0"
DNS="10.64.0.1"
CLIENT_ADDRESS="10.68.x.x/32"
EOF
```

**Configure via UCI (OpenWrt config system):**

```bash
# Add WireGuard interface
uci set network.wg0=interface
uci set network.wg0.proto=wireguard
uci set network.wg0.private_key="$(cat /etc/wireguard/router_private.key)"
uci set network.wg0.addresses="10.68.x.x/32"

# Add VPN peer (your provider's server)
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].public_key="SERVER_PUBLIC_KEY"
uci set network.@wireguard_wg0[-1].endpoint_host="1.2.3.4"
uci set network.@wireguard_wg0[-1].endpoint_port="51820"
uci set network.@wireguard_wg0[-1].route_allowed_ips="1"
uci set network.@wireguard_wg0[-1].allowed_ips="0.0.0.0/0"
uci set network.@wireguard_wg0[-1].persistent_keepalive="25"

uci commit network
/etc/init.d/network restart
```

---

## Configure Firewall and Routing

```bash
# Add WireGuard interface to WAN firewall zone
uci add_list firewall.@zone[1].network="wg0"
uci commit firewall

# Create routing table for VPN traffic
cat >> /etc/iproute2/rt_tables << 'EOF'
100 vpn
EOF

# Route all LAN traffic through VPN
cat > /etc/hotplug.d/iface/30-vpn-route << 'EOF'
#!/bin/sh
if [ "$ACTION" = ifup ] && [ "$INTERFACE" = wg0 ]; then
    ip rule add from 192.168.1.0/24 table vpn
    ip route add default dev wg0 table vpn
fi
EOF
chmod +x /etc/hotplug.d/iface/30-vpn-route

/etc/init.d/firewall restart
```

---

## Verify the VPN Tunnel

```bash
# Check WireGuard status from the router
wg show

# Check the router's public IP (should be VPN server's IP)
curl -s https://am.i.mullvad.net/json

# Or from any device on your network
curl https://ifconfig.me

# Verify DNS isn't leaking
nslookup whoami.akamai.net
```

Expected output from `wg show`:

```
interface: wg0
  public key: ...
  private key: (hidden)
  listening port: xxxxx

peer: SERVER_PUBLIC_KEY
  endpoint: 1.2.3.4:51820
  allowed ips: 0.0.0.0/0
  latest handshake: X seconds ago
  transfer: X MiB received, X MiB sent
```

---

## Split Tunneling (Exclude Specific Devices)

You may want some devices (like a gaming console) to bypass the VPN:

```bash
# Create a separate routing mark for bypass devices
# Get the MAC address of the device to exclude
cat /tmp/dhcp.leases

# Add iptables rule to bypass VPN for specific IP
iptables -t mangle -A PREROUTING -s 192.168.1.105 -j MARK --set-mark 1
ip rule add fwmark 1 table main

# This device uses the normal WAN, others go through VPN
```

---

## OpenVPN Fallback (Older Routers)

If your router doesn't support WireGuard:

```bash
# Install OpenVPN on OpenWrt
opkg install openvpn-openssl luci-app-openvpn

# Download .ovpn config from your VPN provider
# Place it in /etc/openvpn/
cp provider.ovpn /etc/openvpn/

# Start OpenVPN
/etc/init.d/openvpn start
/etc/init.d/openvpn enable
```

OpenVPN is slower than WireGuard (especially on underpowered routers) — expect 20-50% of WireGuard throughput on the same hardware.

---

## Kill Switch (Prevent Traffic if VPN Drops)

```bash
# Block all WAN traffic if WireGuard interface goes down
uci set firewall.@zone[1].input="REJECT"
uci set firewall.@zone[1].output="REJECT"
uci set firewall.@zone[1].forward="REJECT"

# Allow only WireGuard traffic to VPN server on WAN
uci add firewall rule
uci set firewall.@rule[-1].src="lan"
uci set firewall.@rule[-1].dest="wan"
uci set firewall.@rule[-1].dest_ip="1.2.3.4"
uci set firewall.@rule[-1].proto="udp"
uci set firewall.@rule[-1].dest_port="51820"
uci set firewall.@rule[-1].target="ACCEPT"
uci commit firewall
/etc/init.d/firewall restart
```

---

## Related Reading

- [OpenWrt VPN Router WireGuard Setup](/privacy-tools-guide/openwrt-vpn-router-wireguard-setup/)
- [How to Set Up Mullvad VPN on Linux](/privacy-tools-guide/mullvad-vpn-linux-setup-guide/)
- [Network Segmentation for IoT Devices](/privacy-tools-guide/iot-network-segmentation-vlan-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
