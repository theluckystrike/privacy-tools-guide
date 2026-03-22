---
layout: default
title: "How to Detect ARP Spoofing Attacks"
description: "Detect and block ARP spoofing and man-in-the-middle attacks on Linux using arpwatch, XDP filtering, static ARP entries, and dynamic ARP inspection"
date: 2026-03-22
author: theluckystrike
permalink: /detect-arp-spoofing-attacks-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# How to Detect ARP Spoofing Attacks

ARP spoofing lets an attacker on the same network segment silently redirect traffic meant for the gateway through their machine — a classic man-in-the-middle setup. Detection is passive and cheap; prevention requires either network-layer controls or cryptographic alternatives. This guide covers both.

## How ARP Spoofing Works

The Address Resolution Protocol has no authentication. When your machine wants to send a packet to 192.168.1.1, it broadcasts: "Who has 192.168.1.1?" — and trusts the first MAC address that replies. An attacker continuously sends fake ARP replies claiming their MAC is the gateway's address. Your machine updates its ARP cache and starts sending traffic through the attacker.

```
Legitimate:  192.168.1.1  →  AA:BB:CC:DD:EE:FF  (real gateway MAC)
After spoof: 192.168.1.1  →  11:22:33:44:55:66  (attacker's MAC)
```

---

## 1. Inspect the ARP Cache Manually

The fastest sanity check:

```bash
# Show ARP table
arp -n
# or
ip neigh show

# Warning sign: two IP addresses with the same MAC
# or the gateway MAC has changed from what you remember
```

```
192.168.1.1    dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.100  dev eth0 lladdr 11:22:33:44:55:66 REACHABLE
```

If 192.168.1.1 and 192.168.1.100 share the same MAC, one of them is spoofed.

---

## 2. Install and Configure arpwatch

`arpwatch` monitors ARP traffic and emails you when MAC-to-IP bindings change:

```bash
sudo apt install -y arpwatch   # Debian/Ubuntu
sudo dnf install -y arpwatch   # RHEL/Fedora

# Configure mail destination
sudo tee /etc/default/arpwatch > /dev/null <<'EOF'
IFACE="eth0"
ARGS="-u arpwatch -e security@example.com -s 'arpwatch@hostname'"
EOF

# Enable and start
sudo systemctl enable --now arpwatch
sudo journalctl -u arpwatch -f
```

Events arpwatch fires:

| Event | Meaning |
|---|---|
| `new station` | First time this MAC/IP pair was seen |
| `changed ethernet address` | IP moved to a different MAC — possible spoof |
| `flip flop` | MAC bouncing between two IPs — active spoofing |
| `reused old ethernet address` | MAC returned to a previous binding |

A `flip flop` or `changed ethernet address` on your gateway IP is a strong indicator of an active attack.

---

## 3. Capture ARP Traffic with tcpdump

For a quick manual inspection:

```bash
# Capture all ARP packets on the LAN
sudo tcpdump -i eth0 -n arp -w /tmp/arp_capture.pcap

# Read and filter for ARP replies (op 2)
sudo tcpdump -r /tmp/arp_capture.pcap -n arp

# Count ARP replies per source MAC — a high count from one MAC is suspicious
sudo tcpdump -r /tmp/arp_capture.pcap -n arp 2>/dev/null \
  | grep "reply" | awk '{print $NF}' | sort | uniq -c | sort -rn
```

A spoofing tool like `arpspoof` or `ettercap` sends a gratuitous ARP reply roughly once per second. You'll see a flood of replies from the attacker's MAC.

---

## 4. Detect Spoofing with a Python Script

For environments where arpwatch isn't available:

```python
#!/usr/bin/env python3
"""
arp_monitor.py — passive ARP spoof detector using scapy
Run as root: python3 arp_monitor.py eth0
"""
import sys
from collections import defaultdict
from scapy.all import ARP, sniff

IFACE = sys.argv[1] if len(sys.argv) > 1 else "eth0"
ip_to_mac = {}
flip_count = defaultdict(int)

def process_packet(pkt):
    if not pkt.haslayer(ARP):
        return
    arp = pkt[ARP]
    if arp.op != 2:   # only ARP replies
        return
    ip  = arp.psrc
    mac = arp.hwsrc

    if ip in ip_to_mac:
        if ip_to_mac[ip] != mac:
            flip_count[ip] += 1
            print(f"[!] ARP SPOOF ALERT: {ip} changed MAC "
                  f"{ip_to_mac[ip]} → {mac} "
                  f"(flip #{flip_count[ip]})")
    ip_to_mac[ip] = mac

print(f"[*] Monitoring ARP on {IFACE} — Ctrl+C to stop")
sniff(iface=IFACE, filter="arp", prn=process_packet, store=False)
```

```bash
pip3 install scapy
sudo python3 arp_monitor.py eth0
```

---

## 5. Set Static ARP Entries (Prevention)

The most reliable prevention on a small network is to hardcode gateway and critical server MACs:

```bash
# Add a permanent ARP entry
sudo arp -s 192.168.1.1 aa:bb:cc:dd:ee:ff

# Make it permanent — add to /etc/network/interfaces post-up (Debian):
# post-up arp -s 192.168.1.1 aa:bb:cc:dd:ee:ff

# Or via NetworkManager dispatcher
sudo tee /etc/NetworkManager/dispatcher.d/10-static-arp <<'EOF'
#!/bin/bash
if [ "$2" = "up" ]; then
    arp -s 192.168.1.1 aa:bb:cc:dd:ee:ff
fi
EOF
sudo chmod +x /etc/NetworkManager/dispatcher.d/10-static-arp
```

Static entries override any incoming ARP replies. Attackers' spoofed replies will be ignored for protected IPs.

---

## 6. Dynamic ARP Inspection on Managed Switches

If you control the network switch (Cisco IOS / Juniper):

```
! Cisco IOS — Dynamic ARP Inspection
ip arp inspection vlan 10,20

interface GigabitEthernet0/1
 ip arp inspection trust    ! trust uplink to router
 ! all other ports are untrusted by default

! Build DHCP snooping binding table first
ip dhcp snooping
ip dhcp snooping vlan 10,20
```

DAI cross-references ARP packets against the DHCP snooping table. Any ARP reply that doesn't match a valid DHCP lease is dropped at the switch port — before it reaches any host.

---

## 7. Block Spoofed ARP with ebtables (Linux Bridge)

On a Linux bridge (hypervisor, container host):

```bash
# Install ebtables
sudo apt install -y ebtables

# Allow only legitimate ARP replies from the known gateway MAC
sudo ebtables -t filter -A FORWARD -p ARP \
  --arp-op Reply \
  --arp-mac-src ! aa:bb:cc:dd:ee:ff \
  --arp-ip-src 192.168.1.1 \
  -j DROP

# Log suspicious ARP before dropping
sudo ebtables -t filter -A FORWARD -p ARP \
  --arp-op Reply \
  --arp-mac-src ! aa:bb:cc:dd:ee:ff \
  --arp-ip-src 192.168.1.1 \
  -j LOG --log-prefix "ARP-SPOOF: " --log-level 4

sudo ebtables-save > /etc/ebtables.conf
```

---

## 8. Integrate with Snort

Add a Snort local rule for ARP anomalies:

```
# /etc/snort/rules/local.rules
alert arp any any -> any any \
  (msg:"ARP reply flood - possible spoof"; \
   arp.op:2; \
   threshold:type threshold, track by_src, count 20, seconds 5; \
   sid:9000010; rev:1;)
```

---

## Response Checklist

When you detect spoofing:

1. Immediately add a static ARP entry for the gateway: `arp -s <GW_IP> <REAL_MAC>`
2. Identify the attacker's MAC: `ip neigh show` — look for the duplicate MAC
3. Locate the attacker's port on the switch and shut it down
4. Capture ongoing traffic for forensics: `sudo tcpdump -i eth0 host <attacker_IP> -w /tmp/attack.pcap`
5. Check for any established sessions (SSH, database) that may have been intercepted

---

## Related Reading

- [How to Set Up Snort IDS on Linux](/privacy-tools-guide/snort-ids-linux-setup-guide/)
- [How to Use Zeek for Network Monitoring](/privacy-tools-guide/zeek-network-monitoring-guide/)
- [How to Set Up CrowdSec for Server Security](/privacy-tools-guide/crowdsec-server-security-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
