---
layout: default
title: "What to Do If You Find an Unknown Device on Your Network"
description: "A practical guide for developers and power users on identifying, investigating, and securing your network against unknown devices."
date: 2026-03-16
author: theluckystrike
permalink: /what-to-do-if-you-find-unknown-device-on-your-network/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}
Run `arp -a` or `nmap -sn 192.168.1.0/24` to identify the unknown device by MAC address, then look up the manufacturer via the OUI prefix. If unrecognized, block it immediately via your router's MAC filter, change your WiFi password, and check for unauthorized access. Most unknown devices turn out to be forgotten IoT gadgets, but treating every one as a potential threat until identified protects against unauthorized network access.

## Identifying the Unknown Device

The first step is gathering information. Most routers provide a device list through their admin interface, but command-line tools give you more granular control.

### Using ARP to Map Devices

The Address Resolution Protocol (ARP) table maps IP addresses to MAC addresses on your local network:

```bash
# List all devices on your local network
arp -a

# On Linux, you can also use
ip neigh show
```

The output shows active devices with their IP addresses, MAC addresses, and connection state. MAC addresses contain manufacturer prefixes—the first three octets identify the vendor. You can look up these prefixes:

```bash
# Using macvendors.com API (Linux/macOS)
curl "https://api.macvendors.com/$(arp -a | grep '192.168.1.105' | awk '{print $4}')"
```

### Network Scanning Tools

For more comprehensive analysis, use dedicated scanning tools:

```bash
# Install nmap
sudo apt install nmap  # Debian/Ubuntu
brew install nmap      # macOS

# Quick scan of your subnet
nmap -sn 192.168.1.0/24

# Detailed scan with OS detection
sudo nmap -O 192.168.1.0/24
```

The `-sn` flag performs a ping scan without port scanning—useful for discovering all active hosts. The `-O` flag attempts operating system detection, which helps identify device types.

## Investigating the Device

Once you've identified the device's IP and MAC address, investigate further.

### Port Scanning the Unknown Device

Determine what services the device exposes:

```bash
# Scan top 100 ports
nmap 192.168.1.105

# Full port scan
sudo nmap -p- 192.168.1.105

# Scan with service version detection
sudo nmap -sV 192.168.1.105
```

Common findings:
- Port 22 (SSH): Could be a Linux server, Raspberry Pi, or network device
- Port 80/443 (HTTP/HTTPS): Web interface—could be a router, IoT device, or smart appliance
- Port 8080: Often indicates a development server or admin panel
- Port 445 (SMB): File sharing, common on Windows devices

### Checking DHCP Logs

Your router's DHCP logs show lease history, including when devices connected and their hostnames:

```bash
# Many routers expose logs via API or you can query them
# Example: UniFi controller API
curl -k -u admin:password https://192.168.1.1:8443/api/s/default/stat/sta
```

Device hostnames often reveal the type of device. A hostname like "iPhone" or "Galaxy-S21" clearly identifies personal devices.

## Common Scenarios

### Scenario 1: Your Own Forgotten Device

You might find a device you forgot about—a backup Raspberry Pi, an old NAS, or a development board you deployed months ago. Cross-reference the MAC vendor prefix and hostname. If it matches your existing hardware, you can likely whitelist it and move on.

### Scenario 2: Guest Devices

If you run a network with guest access, devices from visitors may appear. Check your router's client list to see if the device is on your guest network segment, which should be isolated from your main network.

### Scenario 3: Neighbors or Wardrivers

In dense housing areas, your WiFi might be reaching neighboring spaces. Ensure WPA3 or strong WPA2 encryption is in place. Check if your router supports client isolation. If the unknown device consistently appears but doesn't match any household members, consider changing your WiFi password.

### Scenario 4: Compromised IoT Device

An unexpected smart device could indicate someone gained physical access or your network was breached. Isolate such devices immediately using VLANs:

```bash
# Example: Creating an IoT VLAN on a Linux bridge
sudo ip link add name br-iot type bridge
sudo ip link set br-iot up
sudo ip link set eth1 master br-iot
```

## Securing Your Network

### Implement Network Segmentation

Separate your network into zones:

- **Trusted**: Computers, phones, servers
- **IoT**: Smart devices, cameras, appliances
- **Guest**: Visitor devices
- **Management**: Router admin interfaces

Most modern routers support guest networks and VLANs. pfSense and OPNsense provide advanced segmentation for power users.

### Enable Network Access Control

Consider deploying 802.1X authentication for enterprise environments:

```bash
# Example: hostapd-wpa3 configuration
interface=wlan0
driver=nl80211
ssid=SecureNet
hw_mode=g
channel=11

# WPA3-Personal
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_psk=your-psk-hash
```

### Set Up Intrusion Detection

Deploy tools like Snort or Suricata for network monitoring:

```bash
# Install Suricata on Ubuntu
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update
sudo apt install suricata

# Run in IDS mode
sudo suricata -c /etc/suricata/suricata.yaml -i eth0
```

### Regular Audits

Schedule weekly or monthly network audits:

```bash
#!/bin/bash
# network-audit.sh
echo "=== Network Audit $(date) ==="
echo "Active devices:"
arp -a | grep -v incomplete
echo "Open ports on gateway:"
nmap -p- 192.168.1.1
```

## Response Protocol

When you find an unknown device, follow this sequence:

1. **Document**: Record IP, MAC, hostname, and discovery time
2. **Isolate**: Disable the device's network access or place it in quarantine VLAN
3. **Investigate**: Run scans to identify services and behavior
4. **Identify**: Cross-reference with known devices and users
5. **Respond**: Allow if legitimate, block if unauthorized
6. **Review**: Check logs for related activity and tighten security

## Conclusion

Network visibility is the foundation of security. By regularly monitoring connected devices and understanding your network's expected behavior, you can quickly identify anomalies. Implement segmentation, enable logging, and maintain an inventory of authorized devices. These practices reduce the attack surface and give you confidence in your network's integrity.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
