---
layout: default
title: "How to Set Up WireGuard with IPv6"
description: "Configure WireGuard VPN with dual-stack IPv4 and IPv6 tunnels, routing all client traffic through a VPS with full leak prevention"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-wireguard-with-ipv6/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Most WireGuard guides only cover IPv4. This leaves an IPv6 leak problem: if your client has native IPv6 connectivity but your VPN tunnel only handles IPv4, traffic to IPv6 addresses bypasses the tunnel entirely and exposes your real IP. This guide sets up WireGuard with both IPv4 and IPv6 routing to prevent leaks.

## Prerequisites

- A VPS with a public IPv4 address
- IPv6 support on the VPS (most cloud providers offer this — check your VPS dashboard)
- WireGuard installed on both server and client
- Ubuntu 22.04 on the server

## Step 1: Enable IPv6 on the VPS

Check if your VPS has an IPv6 address:

```bash
ip -6 addr show
# Look for a global address (not just link-local fe80::)
```

Enable IPv6 forwarding:

```bash
sudo tee -a /etc/sysctl.conf > /dev/null <<'EOF'
# WireGuard IPv4 forwarding
net.ipv4.ip_forward = 1

# WireGuard IPv6 forwarding
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.default.forwarding = 1
EOF

sudo sysctl -p
```

## Step 2: Choose Addressing Scheme

For the WireGuard tunnel network:
- IPv4: `10.100.0.0/24` (server at `10.100.0.1`, clients at `10.100.0.2+`)
- IPv6: `fd42:dead:beef::/64` (ULA — Unique Local Address, globally unique but not publicly routed)
 - Server: `fd42:dead:beef::1`
 - Clients: `fd42:dead:beef::2`, `fd42:dead:beef::3`, etc.

The ULA range (`fd00::/8`) is analogous to RFC1918 for IPv6. Use it for WireGuard tunnel addresses.

## Step 3: Generate Keys

```bash
# Install WireGuard
sudo apt install wireguard

# Server keys
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key

# Client keys (run on client or generate on server and distribute securely)
wg genkey | tee /etc/wireguard/client1_private.key | wg pubkey > /etc/wireguard/client1_public.key

cat /etc/wireguard/server_public.key
cat /etc/wireguard/client1_public.key
```

## Step 4: Server Configuration

Create `/etc/wireguard/wg0.conf` on the server:

```conf
[Interface]
# Server Nebula IPv4 and IPv6 addresses
Address = 10.100.0.1/24, fd42:dead:beef::1/64

# Listen port
ListenPort = 51820

# Server private key
PrivateKey = SERVER_PRIVATE_KEY_HERE

# NAT rules for IPv4 (replace eth0 with your public interface name)
PostUp = iptables -A FORWARD -i %i -j ACCEPT; \
         iptables -A FORWARD -o %i -j ACCEPT; \
         iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# IPv6 NAT (using ip6tables)
# If your VPS has a single global IPv6, use NAT
PostUp = ip6tables -A FORWARD -i %i -j ACCEPT; \
         ip6tables -A FORWARD -o %i -j ACCEPT; \
         ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT; \
           iptables -D FORWARD -o %i -j ACCEPT; \
           iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = ip6tables -D FORWARD -i %i -j ACCEPT; \
           ip6tables -D FORWARD -o %i -j ACCEPT; \
           ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Client 1
PublicKey = CLIENT1_PUBLIC_KEY_HERE
AllowedIPs = 10.100.0.2/32, fd42:dead:beef::2/128
```

Note: If your VPS has a full `/64` IPv6 prefix (not just a single address), you can route client IPv6 addresses directly without NAT. Replace the `ip6tables MASQUERADE` rules with:
```conf
PostUp = ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -A FORWARD -o %i -j ACCEPT
```

And configure each client with a unique address from your delegated prefix.

## Step 5: Enable IPv6 NAT Support in the Kernel

IPv6 NAT (MASQUERADE) requires a kernel module:

```bash
sudo modprobe ip6table_nat
sudo modprobe ip6_tables

# Make it persistent
echo "ip6table_nat" | sudo tee -a /etc/modules-load.d/ip6tables.conf
echo "ip6_tables" | sudo tee -a /etc/modules-load.d/ip6tables.conf
```

Start WireGuard:
```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

## Step 6: Client Configuration

Create `/etc/wireguard/wg0.conf` on the client (Linux) or use the WireGuard app (iOS/Android/Windows):

```conf
[Interface]
# Client's tunnel addresses
Address = 10.100.0.2/32, fd42:dead:beef::2/128

# Client private key
PrivateKey = CLIENT1_PRIVATE_KEY_HERE

# Route ALL traffic (both IPv4 and IPv6) through the tunnel
DNS = 10.100.0.1, fd42:dead:beef::1

[Peer]
PublicKey = SERVER_PUBLIC_KEY_HERE
Endpoint = YOUR_VPS_IP:51820

# Route all IPv4 and IPv6 through tunnel
AllowedIPs = 0.0.0.0/0, ::/0

PersistentKeepalive = 25
```

The key setting is `AllowedIPs = 0.0.0.0/0, ::/0` — this routes all IPv4 AND IPv6 traffic through the tunnel, preventing IPv6 leaks.

Bring the interface up:
```bash
sudo wg-quick up wg0
```

## Step 7: Run a DNS Server for IPv6 (Optional)

If you want to serve DNS over IPv6 from your WireGuard server, use Unbound:

```bash
sudo apt install unbound

sudo tee /etc/unbound/unbound.conf.d/wireguard.conf > /dev/null <<'EOF'
server:
    # Listen on WireGuard interface
    interface: 10.100.0.1
    interface: fd42:dead:beef::1

    # Allow queries from tunnel
    access-control: 10.100.0.0/24 allow
    access-control: fd42:dead:beef::/64 allow

    # Block local queries except from tunnel
    access-control: 0.0.0.0/0 refuse
    access-control: ::/0 refuse

    # Enable IPv6
    do-ip6: yes
    prefer-ip6: no

    # DNSSEC
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
EOF

sudo systemctl enable --now unbound
```

## Step 8: Verify No IPv6 Leaks

```bash
# On the client, check your public IPs
curl -4 https://ifconfig.me   # Should show your VPS IPv4
curl -6 https://ifconfig.me   # Should show your VPS IPv6 (not your ISP's IPv6)

# Check all routes
ip -6 route show
# Should show: default dev wg0 (or via the tunnel)

# DNS leak test
drill @fd42:dead:beef::1 google.com
# Should resolve without hitting your ISP's DNS
```

If `curl -6 https://ifconfig.me` returns your home IPv6, the tunnel is not capturing IPv6 traffic. Check:
1. That `AllowedIPs` includes `::/0`
2. That ip6tables NAT is loaded on the server
3. That your client OS is not routing IPv6 outside the tunnel

## Related Reading

- [How to Set Up WireGuard on VPS for Personal VPN](/privacy-tools-guide/how-to-set-up-wireguard-on-vps-for-personal-vpn/)
- [How to Configure UFW Firewall on Ubuntu](/privacy-tools-guide/how-to-configure-ufw-firewall-on-ubuntu/)
- [How to Set Up Nebula Mesh VPN for Teams](/privacy-tools-guide/how-to-set-up-nebula-mesh-vpn-for-teams/)
- [Set Up a Personal VPN with WireGuard](/privacy-tools-guide/wireguard-personal-vpn-setup-guide)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

## Related Articles

- [How to Use WireGuard for Self-Hosted VPN in 2026](/privacy-tools-guide/articles/how-to-use-wireguard-for-self-hosted-vpn-2026/---)
- [How to Set Up WireGuard on VPS for Personal](/privacy-tools-guide/how-to-set-up-wireguard-on-vps-for-personal-vpn/)
- [Set Up a Personal VPN with WireGuard](/privacy-tools-guide/wireguard-personal-vpn-setup-guide)
- [How to Set Up WireGuard VPN on VPS 2026](/privacy-tools-guide/how-to-set-up-wireguard-vpn-on-vps-2026/)
- [WireGuard Dynamic Endpoint Update](/privacy-tools-guide/wireguard-dynamic-endpoint-update-how-roaming-between-networks-works/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
