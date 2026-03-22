---
layout: default
title: "Network Segmentation for IoT Devices"
description: "Set up VLANs and firewall rules to isolate smart home devices from your main network using OpenWrt, preventing compromised IoT from reaching your computers"
date: 2026-03-22
author: theluckystrike
permalink: /iot-network-segmentation-vlan-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}


Smart bulbs, thermostats, cameras, and voice assistants are computers running outdated firmware with no security update guarantee. Putting them on the same network as your laptops means a compromised smart plug can sniff your banking traffic. VLANs are the standard way to isolate IoT devices — they get internet access but can't reach your main network.

---

## Why IoT Devices Are a Risk

- Most ship with default or weak credentials
- Firmware updates stop within 1-3 years, leaving known vulnerabilities unpatched
- Manufacturers' cloud services may be compromised or shut down
- Some devices phone home to servers in jurisdictions with weak data protection
- A compromised IoT device on your LAN can scan for other devices, exfiltrate data, or pivot to attack your computers

---

## Architecture: Three-Network Model

```
Internet
    |
[Router/Firewall]
    |          |          |
  Main LAN  IoT VLAN  Guest VLAN
  (trusted) (isolated) (isolated)
  PCs       Smart TVs  Visitors
  Phones    Cameras
  NAS       Thermostats
```

**Rules:**
- Main LAN → IoT VLAN: BLOCKED (prevent your PCs from being targeted from IoT, and vice versa)
- IoT VLAN → Main LAN: BLOCKED
- IoT VLAN → Internet: ALLOWED (for cloud functionality)
- IoT VLAN → IoT VLAN: OPTIONAL (needed for device pairing, local control hubs)
- Guest VLAN → Everything else: BLOCKED

---

## Prerequisites

- OpenWrt router (most consumer routers with 16MB+ flash can run OpenWrt)
- Router with managed switch (supports VLAN tagging)
- An access point that supports multiple SSIDs (or a second AP for IoT WiFi)

---

## Step 1: Create the IoT VLAN in OpenWrt

```bash
# SSH into OpenWrt
ssh root@192.168.1.1

# Check existing switch configuration
uci show network | grep switch

# Create VLAN 20 for IoT (tagged on uplink port, untagged on port 3)
uci add network switch_vlan
uci set network.@switch_vlan[-1]=switch_vlan
uci set network.@switch_vlan[-1].device=switch0
uci set network.@switch_vlan[-1].vlan=20
uci set network.@switch_vlan[-1].ports="3 0t"
# Port 0 = CPU (tagged), Port 3 = physical port for IoT wired devices

uci commit network
```

---

## Step 2: Create IoT Network Interface

```bash
# Create interface for IoT VLAN
uci set network.iot=interface
uci set network.iot.ifname=eth0.20
uci set network.iot.proto=static
uci set network.iot.ipaddr=192.168.20.1
uci set network.iot.netmask=255.255.255.0

# Create DHCP server for IoT VLAN
uci set dhcp.iot=dhcp
uci set dhcp.iot.interface=iot
uci set dhcp.iot.start=100
uci set dhcp.iot.limit=150
uci set dhcp.iot.leasetime=12h

# Set DNS for IoT (Pi-hole or AdGuard for telemetry blocking)
uci set dhcp.iot.dhcp_option="6,192.168.20.2"  # IoT VLAN Pi-hole instance

uci commit network dhcp
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
```

---

## Step 3: Create IoT Firewall Zone

```bash
# Add IoT zone to firewall
uci add firewall zone
uci set firewall.@zone[-1].name=iot
uci set firewall.@zone[-1].input=REJECT
uci set firewall.@zone[-1].output=ACCEPT
uci set firewall.@zone[-1].forward=REJECT
uci set firewall.@zone[-1].network=iot

# Allow IoT to access internet
uci add firewall forwarding
uci set firewall.@forwarding[-1].src=iot
uci set firewall.@forwarding[-1].dest=wan

# Allow DHCP and DNS from IoT zone
uci add firewall rule
uci set firewall.@rule[-1].name=iot-dhcp-dns
uci set firewall.@rule[-1].src=iot
uci set firewall.@rule[-1].dest_port="53 67 68"
uci set firewall.@rule[-1].proto="udp"
uci set firewall.@rule[-1].target=ACCEPT

# Block IoT from reaching main LAN (explicit reject rule)
uci add firewall rule
uci set firewall.@rule[-1].name=block-iot-to-lan
uci set firewall.@rule[-1].src=iot
uci set firewall.@rule[-1].dest=lan
uci set firewall.@rule[-1].target=REJECT

uci commit firewall
/etc/init.d/firewall restart
```

---

## Step 4: Create IoT WiFi SSID

```bash
# Add a separate SSID for IoT devices on VLAN 20
uci set wireless.radio0.htmode=HT20

uci add wireless wifi-iface
uci set wireless.@wifi-iface[-1].device=radio0
uci set wireless.@wifi-iface[-1].mode=ap
uci set wireless.@wifi-iface[-1].ssid="HomeIoT"
uci set wireless.@wifi-iface[-1].encryption=psk2
uci set wireless.@wifi-iface[-1].key="iot-network-password"
uci set wireless.@wifi-iface[-1].network=iot
uci set wireless.@wifi-iface[-1].isolate=1  # Prevent IoT devices from talking to each other

uci commit wireless
wifi reload
```

**Note**: `isolate=1` breaks local device discovery (mDNS-based pairing). If you need devices to discover each other (e.g., a Home Assistant hub controlling smart bulbs), you'll need mDNS forwarding — covered below.

---

## Step 5: mDNS/Bonjour Forwarding (Optional)

Some IoT ecosystems use mDNS for device discovery. With strict VLAN isolation, this breaks. You can selectively forward mDNS between VLANs:

```bash
# Install avahi (mDNS responder with multi-subnet support)
opkg install avahi-daemon

cat > /etc/avahi/avahi-daemon.conf << 'EOF'
[server]
use-ipv4=yes
use-ipv6=no
enable-dbus=no

[wide-area]
enable-wide-area=no

[publish]
disable-publishing=yes

[reflector]
enable-reflector=yes
reflect-interfaces=br-lan br-iot
EOF

/etc/init.d/avahi-daemon restart
```

This reflects mDNS packets between `br-lan` and `br-iot` without opening the full firewall between them. Discovery works; direct TCP/UDP connections are still blocked.

---

## Step 6: Assign IoT Devices to the VLAN

**Wired IoT devices**: Connect to the physical port configured for VLAN 20 (port 3 in the example).

**Wireless IoT devices**:
1. Connect the device to the "HomeIoT" SSID
2. It will receive an IP in the 192.168.20.x range
3. Verify isolation:

```bash
# From main LAN, verify you cannot ping IoT devices
ping 192.168.20.101   # Should timeout or be REJECTED

# From IoT device, verify it can reach internet but not main LAN
# (test from the device's built-in network tools, or:)
# Monitor the firewall log
logread -f | grep -i "block-iot"
```

---

## Monitor IoT Traffic

```bash
# Watch what your IoT devices are doing at the network level
tcpdump -i br-iot -nn -v

# Check DNS queries from IoT VLAN to spot suspicious domains
tail -f /var/log/dnsmasq.log | grep "192.168.20"

# See which IoT devices are talking to external IPs
conntrack -L | grep "192.168.20" | awk '{print $5}' | sort | uniq -c | sort -rn
```

---

## Block IoT Telemetry at DNS Level

Even on the isolated VLAN, you can filter outbound DNS to block manufacturer telemetry:

```bash
# Pi-hole on IoT VLAN (or use the same Pi-hole with a separate group)
# Add IoT-specific blocklist to Pi-hole:

# Commonly used IoT telemetry domains:
# iot.samsungcloud.com
# metrics.ring.com
# log-upload.ecobee.com
# analytics.nest.com
# telemetry.simplisafe.com

# In Pi-hole: Groups → Add Group "IoT Devices"
# Add client IPs (192.168.20.0/24) to IoT group
# Assign stricter blocklists to IoT group
```

---

## Verify the Segmentation Works

```python
#!/usr/bin/env python3
# Test script to verify VLAN isolation from a machine on the main LAN
import subprocess, sys

iot_ips = ["192.168.20.101", "192.168.20.102"]  # your IoT device IPs
main_lan_ips = ["192.168.1.50"]                  # your PC

print("Testing IoT isolation...")
for ip in iot_ips:
    result = subprocess.run(
        ["ping", "-c", "1", "-W", "1", ip],
        capture_output=True
    )
    status = "REACHABLE (check firewall!)" if result.returncode == 0 else "BLOCKED (correct)"
    print(f"  {ip}: {status}")

print("\nMain LAN still reachable:")
for ip in main_lan_ips:
    result = subprocess.run(
        ["ping", "-c", "1", "-W", "1", ip],
        capture_output=True
    )
    status = "OK" if result.returncode == 0 else "UNREACHABLE (problem!)"
    print(f"  {ip}: {status}")
```

---

## Related Reading

- [How to Set Up a VPN on Your Router](/privacy-tools-guide/vpn-on-router-setup-guide/)
- [How to Secure Your Smart TV Privacy](/privacy-tools-guide/secure-smart-tv-privacy-guide/)
- [OpenWrt VPN Router WireGuard Setup](/privacy-tools-guide/openwrt-vpn-router-wireguard-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
