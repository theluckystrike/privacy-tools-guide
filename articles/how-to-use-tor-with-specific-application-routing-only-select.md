---
---
layout: default
title: "How To Use Tor With Specific Application Routing Only"
description: "When you need onion routing for specific tasks without routing your entire system through Tor, selective application routing provides the solution. This"
date: 2026-03-16
last_modified_at: 2026-03-16
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

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Selective Tor Routing

The Tor network routes your traffic through at least three relays, encrypting each hop. While this provides strong anonymity, it introduces latency that makes some activities impractical. Selective routing solves this by applying Tor only where you need it.

Several approaches enable application-specific Tor routing:

- **SOCKS proxy configuration**: Configure individual applications to use Tor's SOCKS proxy
- **Linux network namespaces**: Isolate applications in a separate network stack
- **ProxyChains**: Force any application through proxies
- **Virtual machines**: Route specific VMs through Tor

### Step 2: Method 1: Using Tor's SOCKS Proxy Directly

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

### Step 3: Method 2: Linux Network Namespaces

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

### Step 4: Method 3: ProxyChains

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

### Step 5: Method 4: Docker Container Isolation

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

### Step 6: Verify Your Setup

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

### Step 7: Use Cases

Selective Tor routing serves several practical purposes:

- **Development testing**: Test your hidden services locally before deployment
- **Security research**: Anonymize security scanning tools
- **Access control**: Reach .onion versions of services for additional privacy
- **Compartmentalization**: Separate sensitive browsing from regular activities

This setup gives you granular control over which applications receive Tor's anonymity properties while maintaining normal performance for everything else.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to use tor with specific application routing only?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How To Set Up Onion Routing For Email Using Tor Hidden](/privacy-tools-guide/how-to-set-up-onion-routing-for-email-using-tor-hidden-servi/)
- [Tor Hidden Service Setup Guide Developers](/privacy-tools-guide/tor-hidden-service-setup-guide-developers/)
- [Best Browser for Tor Network 2026: A Technical Guide](/privacy-tools-guide/best-browser-for-tor-network-2026/)
- [How to Use Tor Browser Safely](/privacy-tools-guide/tor-browser-safe-usage-guide)
- [How to Use Tor Safely in Country That Criminalizes Its](/privacy-tools-guide/how-to-use-tor-safely-in-country-that-criminalizes-its-use/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
