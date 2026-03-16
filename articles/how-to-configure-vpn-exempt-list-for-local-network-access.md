---
layout: default
title: "How to Configure VPN Exempt List for Local Network Access"
description: "Learn how to configure a VPN exempt list to access local network resources like printers, NAS, and smart home devices while your VPN tunnel remains active."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-vpn-exempt-list-for-local-network-access/
categories: [guides, vpn, networking]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

When you connect to a VPN, all your network traffic routes through an encrypted tunnel to the VPN server. This protects your privacy and security, but it also isolates you from local network resources—printers, NAS devices, smart home hubs, and local development servers become inaccessible. The solution is configuring a VPN exempt list (also called a split-tunnel exclusion list) that tells your VPN client which IP ranges or hosts to access directly, bypassing the tunnel.

This guide covers how to configure VPN exempt lists across common VPN protocols and clients, with practical examples for developers and power users managing local network infrastructure.

## Understanding VPN Exempt Lists

A VPN exempt list specifies IP addresses, CIDR ranges, or hostnames that should not traverse the VPN tunnel. Instead, these destinations are accessed through your local network interface directly. This is the inverse of a split-tunnel include list, which specifies only certain traffic to route through the VPN.

The most common use case is accessing local area network resources:

- Network-attached storage (NAS) devices on `192.168.1.0/24`
- Printers and scanners
- Smart home controllers (Home Assistant, Hubitat)
- Local development servers
- Raspberry Pi or other single-board computers
- Gaming consoles requiring local network discovery

Without an exempt list, your VPN client treats all traffic as VPN-only, effectively placing your entire network behind the VPN server's network topology.

## Configuring Exempt Lists by VPN Type

### WireGuard Exempt Lists

WireGuard uses a simple configuration file where you define `AllowedIPs` for the tunnel and exclude local ranges. The key is setting the correct `Address` and `AllowedIPs` parameters:

```ini
# WireGuard client configuration (wg0.conf)

[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0  # Tunnel all traffic
# Exempt local network - add this line
# Alternative: Use a more specific approach with PostUp/PostDown
```

For true local network exemptions in WireGuard, you have two approaches. The first uses explicit routing with `PostUp` and `PostDown` scripts:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1

# Route local traffic directly, bypassing tunnel
PostUp = ip route add 192.168.1.0/24 via <your-default-gateway> dev eth0
PostDown = ip route del 192.168.1.0/24 via <your-default-gateway> dev eth0

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

Find your default gateway with `ip route | grep default`. The second approach modifies `AllowedIPs` to exclude your local subnet:

```ini
AllowedIPs = 0.0.0.0/0, ::/0, !192.168.1.0/24
```

The `!` operator excludes that range from the tunnel. Note that this syntax works on Linux; macOS and Windows WireGuard clients may handle it differently.

### OpenVPN Exempt Lists

OpenVPN provides more flexible routing options through the `route` and `iroute` directives. To exempt your local network:

```conf
# OpenVPN client configuration

# Exempt the entire 192.168.1.0/24 subnet
route 192.168.1.0 255.255.255.0

# Exempt a specific host
route 192.168.1.100 255.255.255.255

# Exempt multiple subnets
route 10.0.0.0 255.0.0.0
route 172.16.0.0 255.240.0.0
```

Place these directives in your OpenVPN client configuration file (typically `.ovpn` or `.conf`). The `route` directive adds a specific route to the routing table after the tunnel establishes, directing that traffic to your local gateway instead of the VPN.

For more granular control, OpenVPN's `topology` setting affects how routes are assigned. With `topology subnet`, each client receives a subnet rather than a single IP, which can simplify local network access patterns.

### macOS VPN Client (Built-in IKEv2/IPSec)

macOS uses the Network Preferences or command-line `networksetup` for VPN configuration. For the built-in IKEv2 client, you configure excluded networks through the GUI:

1. Open **System Preferences** → **Network**
2. Select your VPN connection
3. Click **Advanced** → **Options**
4. Check **Send all traffic over VPN connection** (disable this for split tunnel)
5. Under **Exclude networks**, add your local subnets

For command-line configuration with `networksetup`:

```bash
# List current VPN configurations
networksetup -listallnetworkservices

# Configure IKEv2 VPN with excluded networks (manual approach)
# This requires creating a custom configuration
```

Note that macOS built-in VPN clients have limited split-tunnel support. Third-party clients like Tunnelblick (OpenVPN) or WireGuard for macOS offer better exempt list functionality.

### Windows VPN Settings

Windows 10 and 11 support split tunneling through PowerShell and the Settings app. For the built-in VPN client:

1. Go to **Settings** → **Network & Internet** → **VPN**
2. Select your VPN connection → **Advanced options**
3. Disable **Send all traffic over VPN connection**

For more control, use PowerShell:

```powershell
# Get current VPN configuration
Get-VpnConnection -Name "YourVPNName"

# Configure split tunneling (excludes specific routes)
Set-VpnConnection -Name "YourVPNName" `
    -SplitTunneling $true `
    -RoutingPolicyType SplitAlways

# Add excluded routes
Add-VpnConnectionRoute -ConnectionName "YourVPNName" `
    -DestinationPrefix 192.168.1.0/24
```

Windows also supports forcing specific applications to bypass the VPN, which is useful for local network tools.

## Common Exempt List Patterns

### Home Network (Typical Consumer Router)

Most home networks use `192.168.0.0/24` or `192.168.1.0/24`:

```
192.168.1.0/24    # Entire local subnet
192.168.1.1/32    # Router/gateway
192.168.1.10-50   # DHCP range (printers, NAS)
```

### Development Network

If you run local servers or containers:

```
10.0.0.0/8        # Private Class A (common for Docker)
172.16.0.0/12     # Private Class B (Docker default bridge)
192.168.99.0/24   # Virtual machine networks
```

### Enterprise/Office Network

Corporate networks often use multiple subnets:

```
10.0.0.0/8        # Main corporate network
10.1.0.0/16      # Office subnet A
10.2.0.0/16      # Office subnet B
172.16.5.0/24    # Printer VLAN
```

## Testing Your Exempt List

After configuring your exempt list, verify it works correctly:

1. **Check routing table**: `ip route` (Linux/macOS) or `route print` (Windows)
2. **Test local access**: Ping a local device—`ping 192.168.1.1`
3. **Confirm VPN path**: Use `traceroute` or `tracert` to verify internet traffic goes through VPN
4. **Check IP address**: Visit a site like whatismyip.com to confirm your public IP shows the VPN server

```bash
# Verify local network is accessible
ping -c 4 192.168.1.1

# Check which interface a local IP uses
ip route get 192.168.1.100

# Confirm VPN is active for internet traffic
curl ifconfig.me
```

## Troubleshooting

If local network resources remain inaccessible:

1. **Verify gateway**: Ensure you're using the correct local gateway IP in your routing rules
2. **Check firewall**: Both your local device and VPN server may have firewall rules blocking access
3. **DNS resolution**: Local hostnames won't resolve over VPN—use IP addresses or configure local DNS
4. **VPN protocol limitations**: Some VPN protocols or providers don't support split tunneling—check with your provider
5. **Overlapping ranges**: If your local network uses the same IP range as the VPN server network, you'll have routing conflicts

## Security Considerations

Exempting local network traffic reduces your attack surface for that traffic, but it also means those requests aren't encrypted by the VPN. Consider:

- Local network traffic is visible to anyone on your local network
- Smart home devices may expose vulnerabilities—keep them updated
- Printers and NAS devices often have weak security—isolate them on VLANs if possible

For maximum security, exempt only the specific hosts you need, not entire subnets. A narrow exempt list reduces exposure while maintaining functionality.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
