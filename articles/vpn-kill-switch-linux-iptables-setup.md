---
layout: default

permalink: /vpn-kill-switch-linux-iptables-setup/
description: "Learn vpn kill switch linux iptables setup with practical examples, tips, and step-by-step instructions for getting the best results."
tags: [privacy-tools-guide, vpn]
author: "Privacy Tools Guide"
reviewed: true
score: 8
date: 2026-03-15
categories: [vpn]
---

layout: default
title: "VPN Kill Switch Configuration on Linux with iptables"
description: "How to configure a VPN kill switch on Linux using iptables and nftables so that all traffic stops if the VPN drops, preventing IP address exposure"
date: 2026-03-21
author: theluckystrike
permalink: /vpn-kill-switch-linux-iptables-setup/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]
---

{% raw %}

A VPN kill switch is a firewall rule that blocks all internet traffic when the VPN connection drops. Without one, your real IP address is exposed the moment the VPN disconnects — even briefly. Many commercial VPN clients include a kill switch, but it often has gaps: it may not activate during initial connection, it may not survive reboots, or it may only block certain traffic types.

Implementing a kill switch at the Linux firewall level gives you control that no VPN client setting can. This guide covers both iptables and nftables implementations.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: How a Kill Switch Works

The logic is simple: allow traffic only through the VPN tunnel interface. Block everything else.

With this rule set:
- Before connecting to VPN: no external traffic is allowed
- While connected: traffic flows normally through the VPN tunnel
- If VPN drops: the tunnel interface disappears, traffic matches the "deny all" rule, nothing leaks

You must know:
1. Your VPN tunnel interface name (usually `tun0`, `wg0`, or `proton0`)
2. Your VPN server's IP address (needed for the initial connection to establish the tunnel)
3. Your local network interface name (usually `eth0` or `wlan0`)

```bash
# Find interface names
ip link show
ip route show
```

### Step 2: Method 1: iptables Kill Switch

### Basic kill switch rules

```bash
#!/bin/bash
# vpn-killswitch.sh
# Replace variables with your actual values

VPN_INTERFACE="wg0"           # Your VPN tunnel interface (wg0, tun0, etc.)
LOCAL_INTERFACE="eth0"         # Your physical network interface
VPN_SERVER_IP="203.0.113.50"  # Your VPN server's IP address
VPN_PORT="51820"               # VPN protocol port (51820=WireGuard, 1194=OpenVPN UDP)
VPN_PROTOCOL="udp"             # udp or tcp

# Flush existing rules
iptables -F
iptables -X

# Default policies: DROP everything
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established/related connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow ONLY the VPN connection itself to pass through the physical interface
iptables -A OUTPUT -o "$LOCAL_INTERFACE" -d "$VPN_SERVER_IP" -p "$VPN_PROTOCOL" --dport "$VPN_PORT" -j ACCEPT

# Allow all traffic through the VPN tunnel interface
iptables -A INPUT -i "$VPN_INTERFACE" -j ACCEPT
iptables -A OUTPUT -o "$VPN_INTERFACE" -j ACCEPT

# Allow local network traffic (optional — comment out for maximum isolation)
iptables -A INPUT -i "$LOCAL_INTERFACE" -s 192.168.0.0/16 -j ACCEPT
iptables -A OUTPUT -o "$LOCAL_INTERFACE" -d 192.168.0.0/16 -j ACCEPT

# Allow DHCPv4 (needed to get an IP from your router)
iptables -A INPUT -i "$LOCAL_INTERFACE" -p udp --sport 67 --dport 68 -j ACCEPT
iptables -A OUTPUT -o "$LOCAL_INTERFACE" -p udp --sport 68 --dport 67 -j ACCEPT

echo "Kill switch enabled. All traffic blocked except VPN tunnel."
```

```bash
chmod +x vpn-killswitch.sh
sudo ./vpn-killswitch.sh
```

### Disable the kill switch

```bash
#!/bin/bash
# vpn-killswitch-disable.sh
iptables -F
iptables -X
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
echo "Kill switch disabled. Normal routing restored."
```

### Persist rules across reboots

```bash
# Install iptables-persistent
sudo apt install iptables-persistent

# Save current rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Rules are automatically loaded on boot by the netfilter-persistent service
sudo systemctl enable netfilter-persistent
```

### Step 3: Method 2: nftables Kill Switch (Modern Linux Systems)

nftables is the replacement for iptables on modern Linux distributions. Debian 10+, Ubuntu 20.04+, Fedora 32+, and Arch all prefer nftables.

```bash
# Check if nftables is active
sudo nft list ruleset
```

Create `/etc/nftables-killswitch.conf`:

```
#!/usr/sbin/nft -f

flush ruleset

define VPN_INTERFACE = wg0
define LOCAL_INTERFACE = eth0
define VPN_SERVER = 203.0.113.50
define VPN_PORT = 51820

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Accept loopback
        iif "lo" accept

        # Accept established connections
        ct state established,related accept

        # Accept traffic from VPN tunnel
        iifname $VPN_INTERFACE accept

        # Accept DHCP
        iifname $LOCAL_INTERFACE udp dport 68 accept

        # Accept local network
        iifname $LOCAL_INTERFACE ip saddr 192.168.0.0/16 accept
    }

    chain output {
        type filter hook output priority 0; policy drop;

        # Accept loopback
        oif "lo" accept

        # Accept traffic through VPN tunnel
        oifname $VPN_INTERFACE accept

        # Accept ONLY VPN connection traffic on physical interface
        oifname $LOCAL_INTERFACE ip daddr $VPN_SERVER udp dport $VPN_PORT accept

        # Accept local network
        oifname $LOCAL_INTERFACE ip daddr 192.168.0.0/16 accept

        # Accept DHCP
        oifname $LOCAL_INTERFACE udp sport 68 dport 67 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }
}
```

Apply the rules:

```bash
sudo nft -f /etc/nftables-killswitch.conf

# To persist: enable nftables service
sudo systemctl enable nftables
sudo cp /etc/nftables-killswitch.conf /etc/nftables.conf
```

### Step 4: Method 3: NetworkManager Kill Switch (for Desktop Users)

If you use NetworkManager and a VPN that supports it:

```bash
# Via nmcli — add a kill switch to an existing VPN connection
nmcli connection modify "MyVPN" ipv4.never-default no
nmcli connection modify "MyVPN" connection.autoconnect-retries -1

# Create a dispatcher script to block traffic when VPN is down
sudo nano /etc/NetworkManager/dispatcher.d/99-vpn-killswitch
```

```bash
#!/bin/bash
# /etc/NetworkManager/dispatcher.d/99-vpn-killswitch

INTERFACE=$1
STATUS=$2

case "$STATUS" in
  vpn-down)
    iptables -I OUTPUT ! -o lo -j REJECT --reject-with icmp-port-unreachable
    ;;
  vpn-up)
    iptables -D OUTPUT ! -o lo -j REJECT --reject-with icmp-port-unreachable
    ;;
esac
```

```bash
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-vpn-killswitch
```

### Step 5: Test the Kill Switch

```bash
# Step 1: Enable kill switch but do NOT connect to VPN
sudo ./vpn-killswitch.sh

# Step 2: Try to reach the internet — should fail
curl -s --max-time 5 https://api.ipify.org
# Should time out or return an error

# Step 3: Connect to VPN
wg-quick up wg0   # For WireGuard
# or: openvpn --config client.ovpn

# Step 4: Verify connectivity through VPN
curl -s https://api.ipify.org
# Should return VPN server's IP, not yours

# Step 5: Simulate VPN drop
sudo ip link set wg0 down

# Step 6: Verify traffic is blocked again
curl -s --max-time 5 https://api.ipify.org
# Should fail — kill switch working correctly
```

### Step 6: IPv6 Leak Prevention

IPv6 traffic can bypass IPv4 kill switch rules. Block it explicitly:

```bash
# Disable IPv6 entirely (simplest approach)
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

# Or block IPv6 with ip6tables kill switch
sudo ip6tables -P INPUT DROP
sudo ip6tables -P OUTPUT DROP
sudo ip6tables -P FORWARD DROP
sudo ip6tables -A INPUT -i lo -j ACCEPT
sudo ip6tables -A OUTPUT -o lo -j ACCEPT
sudo ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# Add VPN IPv6 exceptions if your VPN uses IPv6
```

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Articles

- [How To Test Vpn Kill Switch Actually Works Properly Guide](/privacy-tools-guide/how-to-test-vpn-kill-switch-actually-works-properly-guide/)
- [VPN Kill Switch: How It Works and Which VPNs Have Real](/privacy-tools-guide/vpn-kill-switch-how-it-works-which-vpns-have-real-ones/)
- [How To Use Tcpdump To Verify Vpn Traffic Is Encrypted](/privacy-tools-guide/how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [How to Verify VPN Is Working Correctly 2026](/privacy-tools-guide/how-to-verify-vpn-is-working-correctly-2026/)
- [How VPN Interacts With Firewall Rules Iptables Nftables](/privacy-tools-guide/how-vpn-interacts-with-firewall-rules-iptables-nftables-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

{% endraw %}
