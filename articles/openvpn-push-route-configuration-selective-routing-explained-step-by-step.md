---
layout: default
title: "Openvpn Push Route Configuration Selective Routing Explained"
description: "A guide to configuring OpenVPN push route directives for selective routing, allowing you to control which traffic goes through the VPN"
date: 2026-03-17
author: theluckystrike
permalink: /openvpn-push-route-configuration-selective-routing-explained-step-by-step/
categories: [guides]
tags: [privacy-tools-guide, tools, vpn]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

Configure OpenVPN's push route directive to route only specific traffic through your VPN tunnel instead of all traffic. Use push "route" for selective routing, push "redirect-gateway def1" for full tunneling, or combine both to access local networks while encrypting only sensitive destinations.

## Understanding the Default Behavior

When you establish a standard OpenVPN connection, the server typically pushes a route that redirects all your traffic through the VPN tunnel. This is accomplished using the `redirect-gateway` directive in the server configuration. While this works well for maximum privacy and security, it can cause issues in certain scenarios:

Your local network resources become inaccessible because all traffic is forced through the VPN. Streaming services might not work correctly when they detect VPN traffic. Latency increases for applications that don't need VPN protection. Bandwidth consumption on the VPN server increases unnecessarily.

## The Push Route Directive Explained

The `push` directive in OpenVPN allows the server to send configuration options to clients. The syntax follows this pattern:

```bash
push "route 192.168.1.0 255.255.255.0"
```

This command tells the client to add a route to its routing table after the tunnel is established. The server can push multiple routes to the client, enabling granular control over which networks are accessible through the VPN.

When combined with `redirect-gateway`, the server can push a default route that overrides the client's default gateway:

```bash
push "redirect-gateway def1 bypass-dhcp"
```

The `def1` flag ensures the redirect uses the 0.0.0.0/0 destination with a lower metric, while `bypass-dhcp` prevents the VPN from interfering with DHCP requests on the local network.

## Selective Routing: Send Only Specific Traffic Through VPN

Instead of routing everything through the VPN, you can implement selective routing by pushing specific routes for the networks or services you want to tunnel. This approach gives you precise control over your traffic flow.

### Pushing Specific Subnets

To route only traffic destined for specific networks through the VPN, configure your server to push those exact routes:

```bash
# Server configuration (server.conf)
# Push routes to specific private networks through the tunnel
push "route 10.0.0.0 255.0.0.0"
push "route 172.16.0.0 255.240.0.0"
push "route 192.168.0.0 255.255.0.0"

# Do NOT push redirect-gateway to avoid full tunnel
# This enables split tunneling
```

This configuration ensures that only traffic to private IP ranges flows through the VPN. All other traffic uses your regular internet connection, maintaining full speed and access to local network resources.

### Client-Side Selective Routing

Clients can also implement selective routing using the `route` directive in their client configuration:

```bash
# Client configuration (client.conf)
# Route specific traffic through the VPN
route 10.8.0.0 255.255.255.0
route 172.16.5.0 255.255.255.0

# Exclude specific networks from VPN tunnel
route 192.168.1.0 255.255.255.0 net_gateway
```

The `net_gateway` option tells OpenVPN to use the original default gateway for that specific network, effectively excluding it from the VPN tunnel.

## Step-by-Step: Implementing Selective Routing

### Step 1: Define Your Routing Strategy

Before configuring OpenVPN, determine which traffic should go through the VPN:

- **Full tunnel**: All traffic routes through VPN (use `redirect-gateway`)
- **Split tunnel**: Only specific traffic routes through VPN (push specific routes)
- **Inverse split tunnel**: All except specific traffic routes through VPN (push default route, exclude specific routes client-side)

### Step 2: Configure the Server

Edit your OpenVPN server configuration file:

```bash
# /etc/openvpn/server.conf
# Enable IP forwarding
push "redirect-gateway def1 bypass-dhcp"

# Push specific routes for resources behind the VPN server
push "route 10.8.0.0 255.255.255.0"
push "route 172.16.0.0 255.240.0.0"

# Push DNS servers (important when using VPN for privacy)
push "dhcp-option DNS 10.8.0.1"
push "dhcp-option DNS 8.8.8.8"
```

### Step 3: Configure the Client

For clients that need selective routing, use client-specific configuration:

```bash
# /etc/openvpn/client.conf (or inline in client config)
# Only route traffic to the VPN server's network through the tunnel
route 10.8.0.0 255.255.255.0

# Exclude local network from VPN
route 192.168.1.0 255.255.255.0 net_gateway
```

### Step 4: Verify Your Configuration

After establishing the connection, verify the routing table:

```bash
# On Linux/macOS
route -n

# On Windows
route print

# Check active VPN routes
ip route show
```

Look for entries that include your tunnel interface (usually `tun0` or `tap0`) to confirm proper routing.

## Advanced: Route-Based Access Control

You can combine push routes with firewall rules to create sophisticated access control policies:

```bash
# Server configuration with additional security
push "route 10.8.0.0 255.255.255.0"

# Prevent clients from reaching each other
client-to-client

# Duplicate connections for load balancing
duplicate-cn
```

For environments requiring strict isolation:

```bash
# Block client-to-client communication
push "block-outside-dns"

# Push topology setting for better address management
topology subnet
```

## Troubleshooting Common Issues

### Issue 1: Local Network Inaccessible

If you cannot access local network devices after connecting to VPN, exclude your local subnet from the tunnel:

```bash
# Add to client configuration
route 192.168.1.0 255.255.255.0 net_gateway
```

### Issue 2: DNS Leaks

Ensure DNS queries go through the VPN tunnel:

```bash
# Server pushes secure DNS
push "dhcp-option DNS 10.8.0.1"
push "dhcp-option DNS 1.1.1.1"

# Client configuration to prevent DNS leaks
block-outside-dns
```

### Issue 3: VPN Connection Drops

Implement persistent keepalive to maintain connections through NAT:

```bash
# Add to both server and client configuration
keepalive 10 60
```

## Security Considerations

When implementing selective routing, keep these security principles in mind:

- **Data exposure**: Traffic not routed through VPN is visible to your ISP
- **IP leakage**: Ensure proper firewall rules prevent IP leaks
- **DNS resolution**: Always route DNS queries through the VPN when privacy is critical
- **Split tunneling risks**: Understand the security implications before excluding traffic

For maximum security, use the full tunnel unless you have specific requirements for split tunneling. When using selective routing, regularly audit your routing tables to ensure no sensitive traffic is accidentally leaking outside the VPN.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at zovo.one
