---
---
layout: default
title: "Wireguard Container Setup Docker Network Namespace Isolation"
description: "A practical guide to running WireGuard inside Docker containers with network namespace isolation for enhanced privacy and security"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /wireguard-container-setup-docker-network-namespace-isolation/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Running WireGuard inside Docker provides process isolation while preserving the VPN protocol's performance benefits. This guide covers network namespace isolation, DNS leak prevention, kill switch implementation, and ensuring only designated containers route through the tunnel.

## Table of Contents

- [Why Run WireGuard in Docker?](#why-run-wireguard-in-docker)
- [Prerequisites](#prerequisites)
- [Basic WireGuard Container Setup](#basic-wireguard-container-setup)
- [Network Namespace Isolation](#network-namespace-isolation)
- [Routing Specific Traffic Through the VPN](#routing-specific-traffic-through-the-vpn)
- [Using Docker Compose for Simplified Management](#using-docker-compose-for-simplified-management)
- [Preventing DNS Leaks](#preventing-dns-leaks)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Security Considerations](#security-considerations)
- [Related Reading](#related-reading)

## Why Run WireGuard in Docker?

Running WireGuard as a container offers several advantages. First, you gain process isolation — WireGuard runs in its own container environment, separated from your host system. Second, Docker's networking capabilities make it easy to route specific traffic through the VPN tunnel. Third, containerization simplifies deployment and migration across different systems.

Network namespace isolation takes this further by providing complete network stack separation. When combined with Docker, you can create highly secure and flexible VPN configurations that prevent traffic from other processes from accidentally bypassing the tunnel.

Compared to a native host install, the containerized approach makes it easier to tear down and rebuild your VPN configuration, run multiple WireGuard instances with different endpoints, and audit which processes have tunnel access.

## Prerequisites

Before starting, ensure you have Docker installed and the WireGuard kernel module available on your host:

```bash
# Check if WireGuard module is loaded
lsmod | grep wireguard

# Load the module if not present
sudo modprobe wireguard

# Install wireguard-tools if needed
sudo apt-get install wireguard-tools   # Debian/Ubuntu
sudo pacman -S wireguard-tools         # Arch Linux
sudo dnf install wireguard-tools       # Fedora
```

Your WireGuard config file (`wg0.conf`) should be ready before proceeding, with `[Interface]` and `[Peer]` sections containing your private key, server address, and allowed IPs.

## Basic WireGuard Container Setup

The simplest approach uses the `linuxserver/wireguard` image with the necessary capabilities:

```bash
docker run -d \
  --name wireguard \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/New_York \
  -v /path/to/wg0.conf:/config/wg0.conf:ro \
  -v /lib/modules:/lib/modules:ro \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  --network host \
  linuxserver/wireguard
```

This configuration maps your WireGuard configuration file into the container and grants the necessary capabilities for network interface management. Always mount the configuration read-only (`ro`) to prevent the container from modifying your keys.

The `--network host` mode places the container on the host's network stack. This is the simplest configuration but provides less isolation than the namespace approach below.

## Network Namespace Isolation

For stronger isolation, create a dedicated network namespace. This approach separates the WireGuard interface from the host's default network namespace, giving you precise control over which processes use the VPN tunnel.

### Step 1: Create the Network Namespace

```bash
# Create a new network namespace
sudo ip netns add wg-vpn

# Verify the namespace exists
ip netns list

# Add a loopback interface to the namespace
sudo ip netns exec wg-vpn ip link set lo up
```

### Step 2: Configure the WireGuard Interface in the Namespace

Rather than running the container directly inside the namespace, create the WireGuard interface manually in the namespace and attach it:

```bash
# Create the WireGuard interface
sudo ip link add wg0 type wireguard

# Move it to the namespace
sudo ip link set wg0 netns wg-vpn

# Apply your WireGuard configuration inside the namespace
sudo ip netns exec wg-vpn wg setconf wg0 /path/to/wg0.conf
sudo ip netns exec wg-vpn ip addr add 10.0.0.2/24 dev wg0
sudo ip netns exec wg-vpn ip link set wg0 up
```

### Step 3: Configure Routing in the Namespace

Route all traffic in the namespace through the WireGuard tunnel:

```bash
# Add a default route through the WireGuard peer
sudo ip netns exec wg-vpn ip route add default dev wg0

# Verify routing
sudo ip netns exec wg-vpn ip route show
```

### Step 4: Run Docker Container in the Namespace

Execute containers within the network namespace:

```bash
# Run a container bound to the wg-vpn namespace
sudo ip netns exec wg-vpn docker run -d \
  --name protected-service \
  --network host \
  nginx:alpine
```

Any container launched inside the `wg-vpn` namespace can only reach the internet through the WireGuard tunnel. If the VPN drops, connectivity drops — there is no fallback to the unencrypted host interface.

## Routing Specific Traffic Through the VPN

Rather than routing all traffic through the VPN, you can selectively tunnel specific containers. This approach works well for protecting specific services while maintaining direct access for others.

Create a Docker network that shares the WireGuard container's network stack:

```bash
# Launch the WireGuard container first
docker run -d \
  --name wireguard \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -v /path/to/wg0.conf:/config/wg0.conf:ro \
  -v /lib/modules:/lib/modules:ro \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  linuxserver/wireguard

# Run a container sharing the WireGuard network stack
docker run -d \
  --name protected-app \
  --network container:wireguard \
  nginx:alpine
```

The `--network container:wireguard` option shares the network stack of the WireGuard container. All traffic from `protected-app` passes through the WireGuard interface, including DNS queries.

## Using Docker Compose for Simplified Management

Define your setup in a Docker Compose file for easier management:

```yaml
version: '3.8'

services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./wg0.conf:/config/wg0.conf:ro
      - /lib/modules:/lib/modules:ro
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    networks:
      - wg-net
    restart: unless-stopped

  protected-app:
    image: nginx:alpine
    container_name: protected-app
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
    restart: unless-stopped

networks:
  wg-net:
    driver: bridge
```

Using `network_mode: "service:wireguard"` in Compose is equivalent to `--network container:wireguard` in the CLI. The protected-app container inherits the WireGuard container's entire network stack — including its IP address and routing table.

## Preventing DNS Leaks

DNS leaks are a critical privacy concern in containerized VPN setups. When a container uses the system resolver or Docker's embedded DNS, queries may bypass the tunnel. Configure DNS explicitly in your WireGuard peer section:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 10.0.0.1
MTU = 1420

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

Setting `AllowedIPs = 0.0.0.0/0, ::/0` routes all IPv4 and IPv6 traffic through the tunnel. The `DNS` field tells WireGuard to configure the container's resolver to use the VPN server's DNS, preventing queries from leaking to your ISP.

Verify DNS is not leaking after setup:

```bash
docker exec protected-app nslookup whoami.cloudflare.com 1.1.1.1
```

If the result returns the VPN server's IP rather than your real IP, DNS routes correctly through the tunnel.

## Troubleshooting Common Issues

### Packet Loss and MTU Issues

WireGuard often requires MTU adjustments when running in containers due to the overhead added by the WireGuard encapsulation header (60 bytes for IPv4, 80 for IPv6). Add this to your `[Interface]` section:

```ini
[Interface]
MTU = 1420
```

If you still experience fragmentation issues, lower the MTU further (try 1380) and verify with:

```bash
docker exec wireguard ping -M do -s 1380 8.8.8.8
```

The `-M do` flag prevents fragmentation. If ping fails at 1380 but succeeds at lower sizes, reduce your MTU accordingly.

### Container Connectivity

If your container cannot reach the internet, verify the WireGuard interface is up and the peer is connected:

```bash
# Check WireGuard status in the container
docker exec wireguard wg show

# Check container connectivity
docker exec wireguard ping -c 3 1.1.1.1

# Verify IP routing
docker exec wireguard ip route show
```

A healthy `wg show` output includes the peer's last handshake timestamp (within the last 3 minutes for active connections) and transferred bytes incrementing.

### Kill Switch Implementation

A kill switch prevents traffic from bypassing the VPN if the WireGuard connection drops. Implement it using iptables inside the container:

```bash
# Allow only WireGuard traffic on the physical interface
docker exec wireguard iptables -I OUTPUT -o eth0 -j DROP
docker exec wireguard iptables -I OUTPUT -o eth0 -p udp --dport 51820 -j ACCEPT

# Allow all traffic on the WireGuard interface
docker exec wireguard iptables -I OUTPUT -o wg0 -j ACCEPT
```

These rules block all outgoing traffic on the physical interface except the WireGuard UDP handshake, while allowing all traffic over the WireGuard interface. If the tunnel drops, no traffic escapes.

## Security Considerations

Running WireGuard in Docker adds security layers but requires attention to several details.

Always mount your configuration read-only to prevent accidental modification. Store private keys outside the Docker image — pass them via Docker secrets, not baked into images. Avoid running containers with `--privileged` mode; the `NET_ADMIN` and `SYS_MODULE` capabilities are sufficient and grant far fewer host permissions.

Regularly update your WireGuard image to receive security patches:

```bash
docker pull linuxserver/wireguard:latest
docker compose up -d
```

Container logs can reveal sensitive information including IP addresses and connection metadata. Ensure log aggregation systems handle this data with appropriate access controls.

## Frequently Asked Questions

**Can I run multiple WireGuard tunnels simultaneously in Docker?**
Yes. Run multiple WireGuard containers with different interface names (wg0, wg1, wg2) and different configuration files. Use the container network sharing approach to route different applications through different tunnels.

**Does this setup work on macOS with Docker Desktop?**
Partially. Docker Desktop on macOS runs containers inside a Linux VM. WireGuard kernel module support depends on the VM's kernel. The userspace WireGuard implementation (`wireguard-go`) works cross-platform but has lower throughput than the kernel implementation.

**How do I verify all traffic routes through the VPN?**
From inside the protected container, run `curl ifconfig.me` and compare the returned IP against your VPN server's IP. If they match, all traffic routes through the tunnel. Also run `curl ifconfig.me --interface eth0` — if this returns your real IP, the kill switch is not configured.

**What is the performance overhead of running WireGuard in Docker?**
Container overhead is minimal — typically under 1% throughput reduction compared to a native WireGuard install. The dominant factor is WireGuard's cryptographic overhead (ChaCha20-Poly1305), which is the same whether containerized or native.

## Related Articles

- [Linux Network Namespaces for VPN Isolation](/privacy-tools-guide/linux-network-namespace-vpn-isolation/)
- [OpenWrt VPN Router Setup with WireGuard](/privacy-tools-guide/openwrt-vpn-router-wireguard-setup/)
- [How to Set Up WireGuard on VPS for Personal](/privacy-tools-guide/how-to-set-up-wireguard-on-vps-for-personal-vpn/)
- [How to Use WireGuard for Self-Hosted VPN in 2026](/privacy-tools-guide/articles/how-to-use-wireguard-for-self-hosted-vpn-2026/---)
- [Tor Browser Isolation Container Setup Guide](/privacy-tools-guide/tor-browser-isolation-container-setup-guide/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
