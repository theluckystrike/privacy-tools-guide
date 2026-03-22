---
layout: default
title: "Linux Network Namespaces for VPN Isolation"
description: "Use Linux network namespaces to run specific applications through a VPN tunnel while the rest of your system uses normal routing — no kill switch gaps"
date: 2026-03-21
author: theluckystrike
permalink: /linux-network-namespace-vpn-isolation/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]
---

{% raw %}

A VPN kill switch blocks all traffic when the VPN drops — but it affects your entire system. If you want only certain applications to use the VPN while others use your normal connection, or if you want a guarantee that a specific application can never reach the internet except through the VPN tunnel, network namespaces provide an elegant solution.

A network namespace is a Linux kernel feature that creates an isolated copy of the network stack. A process running inside a namespace sees only the network interfaces that are visible in that namespace — no others. By placing a VPN tunnel inside a namespace and running an application inside that namespace, the application is physically isolated to that tunnel.

## How It Works

The approach:

1. Create a network namespace (`vpn`)
2. Set up a WireGuard or OpenVPN tunnel inside that namespace
3. Run applications inside the namespace — they can only use the tunnel interface
4. If the tunnel is down, there is no fallback route — traffic simply fails

This is a stronger isolation model than a firewall kill switch, because it doesn't rely on firewall rules that can have edge cases. The application process literally has no access to any other network interface.

## Prerequisites

```bash
# Required tools
sudo apt install wireguard iproute2 openresolv

# Verify network namespace support
ip netns help
```

## Step 1: Create the Network Namespace

```bash
# Create a namespace named "vpn"
sudo ip netns add vpn

# Verify
ip netns list
# Output: vpn (id: 0)

# The namespace has no network interfaces yet (except loopback)
sudo ip netns exec vpn ip link show
# Only shows: lo (loopback, initially DOWN)

# Bring up loopback inside the namespace
sudo ip netns exec vpn ip link set lo up
```

## Step 2: Set Up WireGuard Inside the Namespace

Create a WireGuard config file at `/etc/wireguard/wg-vpn.conf`:

```ini
[Interface]
PrivateKey = YOUR_CLIENT_PRIVATE_KEY
Address = 10.67.0.2/32
DNS = 10.64.0.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

Move the WireGuard interface into the namespace after creation:

```bash
# Create a WireGuard interface in the default namespace
sudo ip link add wg-vpn type wireguard

# Move it to the vpn namespace
sudo ip link set wg-vpn netns vpn

# Configure the interface inside the namespace
sudo ip netns exec vpn wg setconf wg-vpn /etc/wireguard/wg-vpn.conf
sudo ip netns exec vpn ip address add 10.67.0.2/32 dev wg-vpn
sudo ip netns exec vpn ip link set wg-vpn up

# The WireGuard handshake needs to reach the VPN endpoint.
# Since the namespace doesn't have internet access yet,
# create a veth pair to link namespace to host for the initial handshake:

# Create virtual ethernet pair
sudo ip link add veth-host type veth peer name veth-ns
sudo ip link set veth-ns netns vpn

# Assign addresses to the veth pair
sudo ip addr add 10.200.200.1/30 dev veth-host
sudo ip netns exec vpn ip addr add 10.200.200.2/30 dev veth-ns

# Bring both up
sudo ip link set veth-host up
sudo ip netns exec vpn ip link set veth-ns up

# Add route inside namespace: use veth for VPN handshake only
# Route VPN endpoint through veth, all else through wg-vpn
VPN_ENDPOINT_IP=$(host vpn.example.com | awk '/has address/ {print $4}')
sudo ip netns exec vpn ip route add "$VPN_ENDPOINT_IP"/32 via 10.200.200.1
sudo ip netns exec vpn ip route add default dev wg-vpn

# Enable IP forwarding on the host for the veth pair
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.200.200.0/30 -j MASQUERADE
```

Verify the tunnel is up:

```bash
sudo ip netns exec vpn wg show
# Should show a recent handshake

sudo ip netns exec vpn curl https://api.ipify.org
# Should show the VPN's IP address
```

## Step 3: Run Applications Inside the Namespace

Any command run with `ip netns exec vpn` uses only the VPN network:

```bash
# Run Firefox through the VPN namespace
sudo ip netns exec vpn sudo -u $USER firefox

# Run a torrent client through the VPN only
sudo ip netns exec vpn sudo -u $USER qbittorrent

# Run curl for a specific network check
sudo ip netns exec vpn curl -s https://api.ipify.org

# Open a shell inside the namespace (all commands in this shell use the VPN)
sudo ip netns exec vpn bash
```

### Run as your regular user (not root)

The `ip netns exec` command requires root. To run applications as your normal user:

```bash
# Method 1: sudo with -u flag
sudo ip netns exec vpn sudo -u $USER application_name

# Method 2: Use unshare for user namespace (more complex setup)
# Method 3: Set up a wrapper script
sudo tee /usr/local/bin/vpn-run << 'EOF'
#!/bin/bash
exec ip netns exec vpn sudo -u $SUDO_USER "$@"
EOF
sudo chmod +x /usr/local/bin/vpn-run

# Usage
sudo vpn-run firefox
sudo vpn-run wget https://example.com/file
```

## Step 4: DNS Inside the Namespace

DNS resolution inside the namespace needs to use the VPN's DNS server:

```bash
# Create a resolv.conf for the namespace
sudo mkdir -p /etc/netns/vpn
echo "nameserver 10.64.0.1" | sudo tee /etc/netns/vpn/resolv.conf

# Linux automatically uses /etc/netns/<namespace>/resolv.conf
# for processes running inside that namespace
```

Test DNS:

```bash
sudo ip netns exec vpn dig google.com
# Should resolve using VPN's DNS server (10.64.0.1 in this example)
```

## Step 5: Automate with a Script

```bash
#!/bin/bash
# /usr/local/bin/vpn-namespace-up
# Sets up the VPN namespace — run once at startup

set -euo pipefail

NS="vpn"
WG_IFACE="wg-vpn"
WG_CONF="/etc/wireguard/wg-vpn.conf"
VPN_DNS="10.64.0.1"

# Create namespace
ip netns add "$NS"
ip netns exec "$NS" ip link set lo up

# Create WireGuard interface in the namespace
ip link add "$WG_IFACE" type wireguard
ip link set "$WG_IFACE" netns "$NS"
ip netns exec "$NS" wg setconf "$WG_IFACE" "$WG_CONF"
ip netns exec "$NS" ip addr add 10.67.0.2/32 dev "$WG_IFACE"
ip netns exec "$NS" ip link set "$WG_IFACE" up

# Create veth pair for VPN handshake
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns "$NS"
ip addr add 10.200.200.1/30 dev veth-host
ip netns exec "$NS" ip addr add 10.200.200.2/30 dev veth-ns
ip link set veth-host up
ip netns exec "$NS" ip link set veth-ns up

# Routing
VPN_EP=$(grep "^Endpoint" "$WG_CONF" | awk '{print $3}' | cut -d: -f1)
VPN_EP_IP=$(dig +short "$VPN_EP" | head -1)
ip netns exec "$NS" ip route add "$VPN_EP_IP"/32 via 10.200.200.1
ip netns exec "$NS" ip route add default dev "$WG_IFACE"

# NAT for initial handshake
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.200.200.0/30 -j MASQUERADE

# DNS
mkdir -p /etc/netns/"$NS"
echo "nameserver $VPN_DNS" > /etc/netns/"$NS"/resolv.conf

echo "VPN namespace ready. Run: ip netns exec $NS [command]"
```

## Cleanup

```bash
# Remove the namespace (and all interfaces in it)
sudo ip netns del vpn
sudo ip link del veth-host
sudo iptables -t nat -D POSTROUTING -s 10.200.200.0/30 -j MASQUERADE
```

## Related Reading

- [VPN Kill Switch Configuration on Linux](/vpn-kill-switch-linux-iptables-setup/)
- [How to Use WireGuard for a Self-Hosted VPN 2026](/articles/how-to-use-wireguard-for-self-hosted-vpn-2026/)
- [OpenWrt VPN Router Setup with WireGuard](/openwrt-vpn-router-wireguard-setup/)
- [AI Tools for Automating Cloud Security Compliance Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-automating-cloud-security-compliance-scanning-i/)
- [AI Container Security Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-container-security-scanning/)

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
