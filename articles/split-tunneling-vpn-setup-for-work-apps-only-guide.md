---
layout: default
title: "Split Tunneling VPN Setup for Work Apps Only Guide"
description: "A practical guide to configuring split tunneling VPN to route only work applications through the VPN while keeping personal traffic on your regular connection."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /split-tunneling-vpn-setup-for-work-apps-only-guide/
categories: [guides]
tags: [vpn, security, privacy, networking]
reviewed: true
score: 8
---

Split tunneling lets you route specific applications through your VPN while keeping other traffic on your regular internet connection. This approach offers significant advantages: you can access corporate resources securely without sacrificing bandwidth for streaming, gaming, or other personal activities. This guide walks you through setting up split tunneling for work applications only.

## Why Split Tunneling Matters

When you use a full VPN tunnel, all your network traffic routes through the VPN server. This means your personal browsing, video calls with family, and streaming services all travel through the corporate VPN infrastructure. The consequences include:

- **Reduced bandwidth** for personal activities
- **Increased latency** on time-sensitive applications like video calls
- **Unnecessary load** on company VPN servers
- **Potential policy violations** if personal usage violates corporate policy

Split tunneling solves these problems by letting you choose which applications use the VPN tunnel and which connect directly to the internet.

## Understanding Split Tunneling Basics

Split tunneling operates at the application or routing level. You have several approaches:

### Application-Based Split Tunneling

Some VPN clients let you select which applications route through the VPN. This is the simplest approach for most users.

### Route-Based Split Tunneling

Advanced users can configure routing tables to direct specific IP ranges through the VPN while everything else uses the default gateway.

### Domain-Based Split Tunneling

Corporate environments often use domain-based rules to route traffic based on domain names or internal hostnames.

## Setting Up Split Tunneling

### WireGuard Configuration

WireGuard uses a simple configuration file. To route only specific applications, you configure the `AllowedIPs` setting:

```ini
[Interface]
PrivateKey = your-private-key
Address = 10.0.0.2/32
DNS = 10.0.0.1

[Peer]
PublicKey = server-public-key
Endpoint = vpn.company.com:51820
AllowedIPs = 10.0.0.0/24, 192.168.100.0/24
```

The `AllowedIPs` setting determines which IP ranges route through the VPN. In this example, only traffic to the corporate network (10.0.0.0/24 and 192.168.100.0/24) goes through the VPN. Everything else uses your regular connection.

To explicitly exclude ranges, use the catch-all approach with specific exclusions:

```ini
AllowedIPs = 0.0.0.0/0
```

Then use routing policy database (RPDB) rules or iptables to exclude specific destinations.

### OpenVPN Configuration

OpenVPN supports split tunneling through the `route` and `iroute` directives. Add these to your client configuration:

```bash
# Route only specific subnets through VPN
route 10.0.0.0 255.255.255.0
route 192.168.100.0 255.255.255.0

# Prevent routing all traffic through VPN
redirect-gateway def1 bypass-dhcp
```

The `redirect-gateway def1 bypass-dhcp` option ensures your default gateway remains unchanged, giving you direct internet access while VPN routes handle corporate subnets.

### Windows Built-in VPN Split Tunneling

Windows 10 and 11 support split tunneling through the Settings app or PowerShell:

```powershell
# Create VPN connection with split tunneling
Add-VpnConnection -Name "Work VPN" `
    -ServerAddress "vpn.company.com" `
    -TunnelType "Automatic" `
    -AuthenticationMethod MSChapv2 `
    -RememberCredential -SplitTunneling $True

# Configure routing for corporate network only
$route = New-NetRoute -DestinationPrefix "10.0.0.0/24" `
    -InterfaceAlias "Work VPN" `
    -NextHop "10.0.0.1"
```

## Configuring Application-Specific Routing

For precise control over which applications use the VPN, you can use network namespaces on Linux or application-specific firewall rules.

### Linux: VPN Exclusively for Specific Apps

Create a dedicated network namespace for work applications:

```bash
# Create network namespace
sudo ip netns add work-vpn

# Set up VPN interface in namespace (WireGuard example)
sudo wg show | sudo ip netns exec work-vpn wg setconf wg0 /etc/wireguard/wg0.conf

# Run application in namespace
sudo ip netns exec work-vpn firefox -P work-profile
```

This approach isolates work applications in their own network stack with the VPN, while all other applications use your regular connection.

### macOS: Per-Application VPN with Third-Party Tools

macOS doesn't natively support application-based split tunneling. Tools like [ Surge](https://nssurge.com/) or [Outline Manager](https://getoutline.org/) provide application-level routing:

```
# Example Surge configuration for split tunneling
[Proxy]
work = ss://your-server-config

[Rule]
PROCESS-NAME, Slack, work
PROCESS-NAME, Microsoft Teams, work
PROCESS-NAME, Outlook, work
DOMAIN-SUFFIX, company.com, work
DOMAIN-SUFFIX, internal.corp, work
GEOIP, CN, work
FINAL, DIRECT
```

### Windows: Force Application Through VPN

Use PowerShell to create a VPN interface and route specific application traffic:

```powershell
# Create a persistent route for work network through VPN
New-NetRoute -DestinationPrefix "10.0.0.0/24" `
    -InterfaceAlias "Work VPN" `
    -NextHop "10.0.0.1" `
    -RouteMetric 1

# Use Windows Filtering Platform to direct app traffic
# This requires creating a filter with specific application ID
```

## Common Split Tunneling Scenarios

### Remote Work Setup

A typical remote worker needs access to:
- Internal corporate network (10.x.x.x, 192.168.x.x)
- Cloud resources (AWS, Azure, GCP)
- Third-party tools (Salesforce, Jira, Confluence)

Everything else—personal email, streaming services, social media—should bypass the VPN.

Example WireGuard configuration:

```ini
[Peer]
PublicKey = server-public-key
AllowedIPs = 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
PersistentKeepalive = 25
```

### Developer Configuration

Developers often need access to:
- Internal development servers
- Container registries
- Cloud provider APIs

```bash
# Route only specific corporate ranges
route 10.0.0.0 255.255.0.0
route 172.16.0.0 255.240.0.0

# Exclude common cloud provider ranges from VPN
# These typically go direct for better performance
```

### Accessing Specific Services Only

For users who only need access to particular resources:

```ini
[Peer]
PublicKey = server-public-key
AllowedIPs = vpn.company.com/32, gitlab.internal/32
```

This routes only traffic to the specified hostname (via DNS resolution) and IP through the VPN.

## Troubleshooting Split Tunneling

### Traffic Not Routing Correctly

Verify your routing table:

```bash
# Linux
ip route show

# Windows
route print

# macOS
netstat -rn
```

Check that your corporate network routes use the VPN interface while default traffic uses your regular gateway.

### DNS Leaks

When using split tunneling, DNS requests might leak to your ISP. Force DNS through the VPN tunnel:

```bash
# Linux - force DNS through VPN
sudo resolvectl dns wg0 10.0.0.1
sudo resolvectl routing-domain wg0 private

# Verify
resolvectl query vpn.company.com
```

### Application Connectivity Issues

Some applications hardcode server addresses. Use `tcpdump` or Wireshark to verify traffic flow:

```bash
# Capture traffic on VPN interface
sudo tcpdump -i wg0 -n

# Capture traffic on regular interface
sudo tcpdump -i eth0 -n
```

Compare the captured traffic to ensure corporate IP ranges go through the VPN interface.

## Security Considerations

Split tunneling introduces security considerations:

### Data Exfiltration Risk

With split tunneling, malicious software on your machine could potentially access both corporate resources and the internet simultaneously. Mitigate this with:

- Endpoint detection and response (EDR) software
- Application allowlisting
- Network segmentation

### Split Tunnel Detection

Some organizations detect split tunneling through:

- Network traffic analysis
- Client health checks
- VPN server logs

Ensure your configuration aligns with corporate policy.

### Always-On VPN Considerations

For high-security environments, consider combining split tunneling with always-on VPN:

```bash
# Linux - kill switch using iptables
sudo iptables -A OUTPUT ! -o wg0 -j DROP
sudo ip6tables -A OUTPUT ! -o wg0 -j DROP
```

This prevents any traffic when the VPN is down.

## Measuring Split Tunneling Effectiveness

Test your configuration to ensure it's working as expected:

```bash
# Check public IP (should show your ISP IP, not VPN)
curl ifconfig.me

# Check corporate network routing (should show VPN endpoint)
ip route get 10.0.0.1

# Verify DNS resolution
nslookup gitlab.internal
```

## Best Practices

1. **Start simple**: Use your VPN client's built-in split tunneling before custom configurations
2. **Document your setup**: Note which apps and domains route through VPN
3. **Test regularly**: Verify split tunneling after system updates
4. **Monitor bandwidth**: Ensure personal activities don't consume corporate VPN resources
5. **Coordinate with IT**: Ensure compliance with organizational security policies

## Conclusion

Split tunneling provides the best of both worlds: secure access to corporate resources without sacrificing performance for personal activities. Whether you use WireGuard's elegant routing configuration, OpenVPN's flexible options, or application-specific tools, the key is identifying which traffic genuinely needs VPN protection and which can safely travel direct.

Start with your VPN client's built-in split tunneling options, then graduate to custom configurations as your needs become more specific. Remember to test thoroughly and maintain documentation for troubleshooting.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
