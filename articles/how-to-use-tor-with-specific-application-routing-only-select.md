---
layout: default
title: "How To Use Tor With Specific Application Routing Only Select"
description: "Learn how to route only specific applications through Tor while keeping other traffic on your regular internet connection. Practical setup guide for."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-tor-with-specific-application-routing-only-select/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

When you need onion routing for specific tasks without routing your entire system through Tor, selective application routing provides the solution. This approach lets you route only designated applications through the Tor network while maintaining direct connections for everything else. Developers and power users commonly need this setup for tasks like testing Tor-hidden services, accessing .onion domains, or anonymizing particular application traffic without sacrificing performance on other operations.

This guide covers multiple methods to achieve per-application Tor routing on Linux, macOS, and Windows systems.

## Understanding Selective Tor Routing

The Tor network routes your traffic through at least three relays, encrypting each hop. While this provides strong anonymity, it introduces latency that makes some activities impractical. Selective routing solves this by applying Tor only where you need it.

Several approaches enable application-specific Tor routing:

- **SOCKS proxy configuration**: Configure individual applications to use Tor's SOCKS proxy
- **Linux network namespaces**: Isolate applications in a separate network stack
- **ProxyChains**: Force any application through proxies
- **Virtual machines**: Route specific VMs through Tor

## Method 1: Using Tor's SOCKS Proxy Directly

Tor provides a SOCKS proxy on localhost port 9050 by default. Many applications support SOCKS proxy configuration natively.

### Starting Tor with SOCKS Proxy

First, ensure Tor is installed and running:

```bash
# Install Tor
sudo apt install tor        # Debian/Ubuntu
sudo pacman -S tor         # Arch Linux
brew install tor           # macOS

# Start Tor service
sudo systemctl start tor   # Linux
tor &                      # macOS/standalone
```

Configure Tor to allow SOCKS connections:

```bash
# Edit /etc/tor/torrc
SOCKSPort 9050
SOCKSPort 9051 # For applications needing separate port
AllowUnverifiedNodes false
```

### Configuring Applications

**cURL through Tor:**

```bash
curl --socks5 localhost:9050 https://check.torproject.org/api/ip
```

**SSH through Tor:**

```bash
ssh -o ProxyCommand='nc -X 5 -x localhost:9050 %h %p' user@onion-service
```

**Git through Tor:**

```bash
git clone git@github.com:user/repo.git
git config core.gitproxy "nc -X 5 -x localhost:9050"
```

For Git over Tor hidden services, use:

```bash
git clone socks5://localhost:9050/[onion-address]/repo.git
```

## Method 2: Linux Network Namespaces

Network namespaces provide stronger isolation by giving applications their own network stack. This method completely separates Tor traffic from your main network.

### Setup Network Namespace with Tor

```bash
# Create namespace
sudo ip netns add tor-net

# Create veth pair
sudo ip link add veth0 type veth peer name veth1

# Assign peer to namespace
sudo ip link set veth1 netns tor-net

# Configure namespace network
sudo ip netns exec tor-net ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec tor-net ip link set veth1 up
sudo ip netns exec tor-net ip link set lo up

# Configure main namespace
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up

# Setup NAT for namespace internet access via Tor
sudo iptables -A FORWARD -i veth0 -o veth1 -j ACCEPT
sudo iptables -A FORWARD -i veth1 -o veth0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE

# Start Tor in namespace with proper interface binding
sudo ip netns exec tor-net tor \
  --SOCKSPort 9050 \
  --TransPort 9040 \
  --DNSPort 5353 \
  --AutomapHostsOnResolve 1 \
  --VirtualAddrNetworkIPv4 10.192.0.0/10 \
  -f /etc/tor/torrc
```

### Running Applications in Namespace

```bash
# Run any command through Tor namespace
sudo ip netns exec tor-net curl https://check.torproject.org/api/ip
sudo ip netns exec tor-net firefox
sudo ip netns exec tor-net python3 script.py
```

This approach ensures complete network isolation—no DNS leaks, no IPv6 escapes, nothing leaves the namespace except through Tor.

## Method 3: ProxyChains

ProxyChains forces any application through SOCKS proxies without application-level configuration.

### Installation and Configuration

```bash
# Install ProxyChains
sudo apt install proxychains4
# or on macOS
brew install proxychains4
```

Configure ProxyChains:

```bash
# Edit /etc/proxychains4.conf
# Add at the end:
socks5  127.0.0.1  9050
```

### Usage

```bash
# Run commands through Tor
proxychains4 curl https://check.torproject.org/api/ip
proxychains4 nmap -sT scanme.nmap.org
proxychains4 python3 requests.py
```

ProxyChains works with most TCP-based applications but has limitations with applications that perform their own socket handling.

## Method 4: Docker Container Isolation

Docker provides another isolation layer for Tor-routed applications.

### Running Applications in Tor-Enabled Container

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM alpine:latest
RUN apk add --no-cache tor curl
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
EOF

# Create entrypoint
cat > entrypoint.sh << 'EOF'
#!/bin/sh
tor &
sleep 5
exec "$@"
EOF

# Build and run
docker build -t tor-app .
docker run -it tor-app curl --socks5 localhost:9050 https://check.torproject.org/api/ip
```

## Verifying Your Setup

Always verify that traffic actually routes through Tor:

```bash
# Check IP through Tor
curl --socks5 localhost:9050 https://check.torproject.org/api/ip

# Verify with multiple checks
curl --socks5 localhost:9050 https://ipinfo.io/json
curl --socks5 localhost:9050 https://api.my-ip.io/v2/ip

# Check DNS resolution
dig +short myip.opendns.com @resolver1.opendns.com
```

Your Tor IP should differ from your regular IP, and repeated checks should show different exit nodes.

## Common Issues and Solutions

**Application doesn't support SOCKS:**

Use ProxyChains or run in a network namespace. Many CLI tools support `--proxy` flags or environment variables like `ALL_PROXY`.

**DNS leaks:**

Configure applications to use Tor's DNS resolver (port 5353). In network namespace setup, all DNS automatically goes through Tor.

**IPv6 leaks:**

Disable IPv6 entirely or ensure your application only makes IPv4 connections. Tor supports IPv6 but many applications handle it poorly.

**Connection timeouts:**

Tor adds significant latency. Increase connection timeouts in your applications. Some services may block Tor exit nodes entirely.

## Use Cases

Selective Tor routing serves several practical purposes:

- **Development testing**: Test your hidden services locally before deployment
- **Security research**: Anonymize security scanning tools
- **Access control**: Reach .onion versions of services for additional privacy
- **Compartmentalization**: Separate sensitive browsing from regular activities

This setup gives you granular control over which applications receive Tor's anonymity properties while maintaining normal performance for everything else.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
