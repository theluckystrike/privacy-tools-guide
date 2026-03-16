---
layout: default
title: "How to Configure DNS Over HTTPS Inside VPN Tunnel"
description: "A practical guide for developers and power users to configure DNS over HTTPS within a VPN tunnel. Learn methods, code examples, and troubleshooting."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-dns-over-https-inside-vpn-tunnel/
categories: [guides]
---

{% raw %}

DNS over HTTPS (DoH) encrypts your DNS queries, preventing eavesdropping and manipulation. Running DoH inside a VPN tunnel adds an extra layer of privacy by ensuring all DNS resolution happens through the encrypted tunnel. This guide covers multiple methods to configure DoH within your VPN setup, targeting developers and power users who want granular control over their network traffic.

## Why Run DoH Inside a VPN Tunnel

When you use a VPN without explicit DoH configuration, DNS queries often leak outside the encrypted tunnel. Your ISP or VPN provider handles DNS resolution, which means they can see which domains you access even if the traffic itself is encrypted. By configuring DoH inside the tunnel, you route DNS queries through encrypted HTTPS to a DoH provider of your choice, completely bypassing traditional DNS infrastructure.

This approach provides defense in depth: the VPN encrypts your traffic, and DoH encrypts your DNS queries. Neither your ISP nor your VPN provider can see which domains you're resolving.

## Prerequisites

Before configuring DoH inside your VPN tunnel, ensure you have:

- A working VPN client (OpenVPN, WireGuard, or strongSwan)
- Administrative access to your VPN client or server
- A DoH provider (Cloudflare, Google, Quad9, or a self-hosted resolver)
- Basic understanding of networking concepts (DNS, UDP/TCP ports, tunnel interfaces)

## Method 1: Configure DoH on Your VPN Client

The simplest approach involves configuring your VPN client to use DoH instead of the default DNS servers. This works well for WireGuard and OpenVPN clients that support DNS configuration.

### WireGuard Configuration

WireGuard allows you to specify DNS servers in the peer configuration. Add the DoH resolver address to your WireGuard config file:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

However, this basic configuration still uses standard DNS over UDP port 53. To use DoH, you need a tool that intercepts DNS queries and forwards them over HTTPS. Enter `dnscrypt-proxy`, which you can run alongside your VPN client.

Install dnscrypt-proxy and configure it to listen on a local address, then point your WireGuard DNS to that local resolver:

```bash
# /etc/dnscrypt-proxy/dnscrypt-proxy.toml
listen_addresses = ['127.0.0.1:53']
bootstrap_resolvers = ['1.1.1.1:53']
doh_sources = ['https://cloudflare-dns.com/dns-query']
```

Then update your WireGuard config to use `127.0.0.1` as the DNS server. Now all DNS queries from your VPN client go through the local dnscrypt-proxy, which encrypts them via DoH.

### OpenVPN Configuration

OpenVPN supports the `dhcp-option DNS` directive to push DNS servers to clients. For DoH, you need a client-side resolver. Configure your OpenVPN client to use a local DoH proxy:

```bash
# /etc/openvpn/client.conf
client
dev tun
remote vpn.example.com 1194
proto udp
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256

# Redirect all traffic through VPN
redirect-gateway def1 bypass-dhcp

# Push DoH resolver to client (requires client-side setup)
push "dhcp-option DNS 10.0.0.1"
```

On your client machine, run dnscrypt-proxy listening on `10.0.0.1`. This intercepts DNS queries from the VPN and encrypts them via DoH.

## Method 2: Configure DoH on the VPN Server

For more centralized control, configure DoH on your VPN server. This approach ensures all clients automatically use DoH without individual configuration.

### strongSwan with DoH Forwarding

If you use strongSwan for IPsec VPNs, you can set up a local DNS forwarder that handles DoH. Install `unbound` or `bind9` and configure it to forward DNS queries to DoH providers:

```conf
# /etc/unbound/unbound.conf.d/doh-forward.conf
server:
    interface: 0.0.0.0
    access-control: 10.0.0.0/24 allow
    forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-addr: 1.1.1.1@853#cloudflare-dns.com
        forward-addr: 8.8.8.8@853#dns.google
```

Configure your strongSwan IPsec configuration to assign this DNS server to clients:

```conf
# /etc/ipsec.conf
conn myvpn
    leftsubnet=0.0.0.0/0
    right=%any
    rightsubnet=10.0.0.0/24
    authby=secret
    auto=add
    esp=aes256gcm16
    ike=aes256gcm16-prfsha256-ecp256
    mobike=no

# Push DNS server to clients
rightdns=10.0.0.1
```

Now all VPN clients receive the DoH-configured DNS server automatically.

### WireGuard VPN Server with systemd-resolved

On Linux servers, you can integrate WireGuard with systemd-resolved to handle DoH. Create a WireGuard interface and configure systemd-resolved to use DoH:

```bash
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 8.8.8.8
DNSOverTLS=cloudflare
Domains=~.
```

Configure your WireGuard server to assign this DNS to clients:

```ini
[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.0.0.2/32
DNS = 10.0.0.1
```

The `DNSOverTLS=cloudflare` setting ensures all DNS queries from the server go through DoH, and clients inherit this configuration.

## Method 3: Split Tunneling with DoH

For scenarios where you want only specific traffic through the VPN while maintaining DoH, split tunneling provides flexibility. Configure your VPN client to route specific subnets through the tunnel while using DoH for all DNS queries.

WireGuard's `AllowedIPs` directive enables split tunneling:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 127.0.0.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
# Route only specific networks through VPN
AllowedIPs = 10.0.0.0/8, 192.168.100.0/24
PersistentKeepalive = 25
```

Run dnscrypt-proxy locally to handle DoH for all DNS queries, regardless of whether traffic goes through the VPN. This ensures DNS privacy even for traffic that doesn't use the VPN tunnel.

## Verifying Your Configuration

After configuring DoH inside your VPN tunnel, verify the setup works correctly:

1. **Check DNS resolution**: Run `dig example.com` or `nslookup example.com` and confirm responses come back
2. **Verify DoH is active**: Use `tcpdump -i any -n port 53` to confirm no plain DNS traffic leaves your machine
3. **Test DoH encryption**: Use a tool like `dog` or `curl` to query your DoH provider directly:

```bash
curl -H 'accept: application/dns-json' 'https://cloudflare-dns.com/dns-query?name=example.com&type=A'
```

4. **Check VPN DNS assignment**: On Linux, run `resolvectl status` to see which DNS servers are active

## Troubleshooting

Common issues and solutions:

- **DNS leaks**: Ensure your VPN client configuration explicitly sets DNS servers and that no other interfaces override this
- **DoH connection failures**: Verify firewall rules allow outbound HTTPS (port 443) for DoH providers
- **VPN connection drops**: Check that `PersistentKeepalive` is set in WireGuard configurations to maintain NAT mappings
- **Client DNS not updating**: Restart the VPN service and clear local DNS caches with `systemd-resolve --flush-caches`

## Conclusion

Configuring DNS over HTTPS inside a VPN tunnel significantly enhances your privacy posture. Whether you choose client-side configuration, server-side forwarding, or split tunneling with DoH, the key is ensuring DNS queries never escape unencrypted. The methods outlined here provide flexibility for different VPN protocols and deployment scenarios, allowing developers and power users to implement defense in depth for their network traffic.

For most users, running dnscrypt-proxy locally alongside the VPN client provides the best balance of simplicity and effectiveness. Advanced users managing their own VPN servers may prefer server-side configuration for centralized control.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
