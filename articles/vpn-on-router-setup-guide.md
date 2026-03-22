---
layout: default
title: "How to Set Up a VPN on Your Router"
description: "Configure WireGuard or OpenVPN on your router with OpenWrt or DD-WRT to protect every device on your network without per-device clients"
date: 2026-03-22
author: theluckystrike
permalink: /vpn-on-router-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}


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

## DNS Leak Prevention

Even with the VPN tunnel active, DNS queries can bypass the tunnel and reveal your browsing activity to your ISP. This is the most common privacy failure in router VPN setups.

```bash
# Force all DNS through the VPN tunnel on OpenWrt
# Set the router's DNS to your VPN provider's DNS
uci set dhcp.@dnsmasq[0].server="10.64.0.1"  # Mullvad DNS
uci set dhcp.@dnsmasq[0].noresolv="1"
uci commit dhcp
/etc/init.d/dnsmasq restart

# Block DNS queries that try to leave on port 53 via WAN
iptables -I FORWARD -i br-lan -o wan -p udp --dport 53 -j REJECT
iptables -I FORWARD -i br-lan -o wan -p tcp --dport 53 -j REJECT
```

Verify there are no leaks after configuration:

```bash
# From a device on your LAN, visit https://dnsleaktest.com
# Or test via command line from a LAN device:
nslookup whoami.akamai.net
dig +short myip.opendns.com @resolver1.opendns.com

# All answers should show the VPN server's IP, not your real IP
```

For providers that support DNS over HTTPS (DoH), configure it at the router level using `https-dns-proxy`:

```bash
opkg install https-dns-proxy luci-app-https-dns-proxy
uci set https-dns-proxy.@https-dns-proxy[0].resolver_url="https://dns.mullvad.net/dns-query"
uci commit https-dns-proxy
/etc/init.d/https-dns-proxy restart
```

## Choosing a VPN Provider for Router Use

Not every VPN provider works well at the router level. Key requirements:

| Requirement | Why It Matters |
|-------------|----------------|
| WireGuard support | Native performance; OpenVPN is 30-50% slower |
| Multiple simultaneous connections | Router counts as one device — all LAN devices share it |
| Port forwarding (optional) | Required for hosting services behind the VPN |
| Static IP support | Useful for allowlisting the VPN exit IP in services |
| No-logs policy (audited) | Third-party audits matter more than marketing claims |

**Mullvad** works well for router use: WireGuard config files downloadable per-server, audited no-logs policy, anonymous account numbers (no email required). Download the WireGuard config at `mullvad.net/en/account/wireguard-config`.

**ProtonVPN** provides WireGuard configs for router use and supports Secure Core (routing through Switzerland or Iceland before the VPN exit). The ProtonVPN app cannot be installed on a router, but the raw WireGuard config can.

**IVPN** is another audited option with WireGuard support and multi-hop configurations that route traffic through two VPN servers before exiting.

Avoid providers that only offer proprietary apps without WireGuard config download — those cannot be used on OpenWrt.

## Performance Tuning

Router CPUs are significantly weaker than desktop CPUs. WireGuard's ChaCha20 cipher is hardware-friendly, but you may still see throughput limitations on budget routers.

```bash
# Check current WireGuard throughput
wg show wg0 transfer

# Test real-world speeds via the router
# SSH into router and run:
wget -O /dev/null https://speedtest.net/  # basic test

# For ARM routers, enable hardware crypto acceleration if available
opkg install kmod-crypto-hw-tegra kmod-crypto-cbc
```

Expected throughput on common hardware:
- **GL.iNet GL-AX1800** (ARM, WiFi 6): ~200 Mbps WireGuard
- **GL.iNet GL-MT1300** (MIPS): ~50-80 Mbps WireGuard
- **Raspberry Pi 4 as router**: ~300+ Mbps WireGuard

If your ISP provides more than 200 Mbps and you have a MIPS-based router, the router CPU is your bottleneck. Either upgrade hardware or accept that the VPN limits your throughput.

```bash
# Monitor router CPU during speed test
top  # watch CPU usage on the router SSH session
# If CPU hits 100% during transfer, hardware is the limit
```

For high-throughput setups, the **PC-based pfSense or OPNsense** approach with an AES-NI capable CPU and WireGuard provides near-wire speeds regardless of ISP bandwidth.

## Maintaining the VPN Configuration

Router VPN configurations require periodic maintenance:

**Rotate WireGuard keys periodically.** Most VPN providers allow generating new keys without changing your account. Generate new keys every 3-6 months:

```bash
# Generate new key pair
wg genkey | tee /etc/wireguard/new_private.key | wg pubkey > /etc/wireguard/new_public.key

# Update your VPN account with the new public key
# Then update the UCI config:
uci set network.wg0.private_key="$(cat /etc/wireguard/new_private.key)"
uci commit network
/etc/init.d/network restart
```

**Check for OpenWrt security updates:**

```bash
# Check available package updates
opkg update && opkg list-upgradable

# Upgrade all packages
opkg upgrade $(opkg list-upgradable | awk '{print $1}')
```

**Test the kill switch after every firmware update.** Firmware updates can reset firewall rules. After any OpenWrt upgrade, verify the kill switch still blocks non-VPN traffic by temporarily disabling the WireGuard interface and confirming that LAN devices lose internet access entirely.

## Related Reading

- [OpenWrt VPN Router WireGuard Setup](/privacy-tools-guide/openwrt-vpn-router-wireguard-setup/)
- [How to Set Up Mullvad VPN on Linux](/privacy-tools-guide/mullvad-vpn-linux-setup-guide/)
- [Network Segmentation for IoT Devices](/privacy-tools-guide/iot-network-segmentation-vlan-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)


## Frequently Asked Questions


**How long does it take to set up a vpn on your router?**

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
