---
layout: default
title: "WireGuard PostUp/PostDown Scripts for Advanced Routing"
description: "Master WireGuard PostUp and PostDown scripts for advanced routing. Learn to implement split tunneling, kill switches, custom DNS, and selective traffic"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /wireguard-postup-postdown-scripts-for-advanced-routing-confi/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, advanced]
---

{% raw %}

WireGuard is renowned for its simplicity and performance, but many users overlook powerful features like PostUp and PostDown scripts. These hooks let you execute shell commands automatically when a VPN tunnel activates or deactivates, enabling advanced routing configurations that go far beyond basic point-to-point connections.

## Understanding PostUp and PostDown

WireGuard configuration files support two optional directives: `PostUp` and `PostDown`. The system executes `PostUp` commands after the interface comes up, and `PostDown` commands before the interface tears down. This timing makes them ideal for setting up complex routing rules, managing firewall changes, or automating network adjustments.

The syntax is straightforward:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/24
PostUp = /path/to/script.sh
PostDown = /path/to/script-down.sh

[Peer]
PublicKey = <peer-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
```

You can also inline commands directly:

```ini
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT
```

The `%i` variable represents the interface name (typically `wg0`), making your scripts portable across different configurations.

## Implementing Split Tunneling with Custom Routes

Split tunneling lets you route only specific traffic through the VPN while keeping other traffic on your regular connection. PostUp scripts excel at this use case.

Suppose you want only traffic to your corporate network (10.0.0.0/8) to go through the VPN:

```ini
[Interface]
PrivateKey = <key>
Address = 10.2.0.5/24

[Peer]
PublicKey = <peer-key>
Endpoint = corporate.vpn.com:51820
AllowedIPs = 10.0.0.0/8, 172.16.0.0/12
PersistentKeepalive = 25
```

For more dynamic control, use a PostUp script:

```bash
#!/bin/bash
# /etc/wireguard/scripts/split-tunnel-setup.sh

# Add route to corporate network via VPN
ip route add 10.0.0.0/8 dev wg0 table 100
ip route add 172.16.0.0/12 dev wg0 table 100

# Create routing policy for specific traffic
ip rule add from 10.2.0.5 lookup 100 prio 1000
```

The corresponding PostDown script cleans these up:

```bash
#!/bin/bash
# /etc/wireguard/scripts/split-tunnel-teardown.sh

ip rule del from 10.2.0.5 lookup 100 prio 1000
ip route del 10.0.0.0/8 dev wg0 table 100
ip route del 172.16.0.0/12 dev wg0 table 100
```

This approach gives you granular control over which subnets use the VPN tunnel.

## Building a Kill Switch

A kill switch prevents data leaks by blocking all non-VPN traffic when the tunnel drops. Combine PostUp and PostDown with iptables to implement this:

```ini
[Interface]
PrivateKey = <key>
Address = 10.2.0.5/24

[Peer]
PublicKey = <peer-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0

PostUp = iptables -I INPUT -i wg0 -j ACCEPT; \
 iptables -I FORWARD -i wg0 -j ACCEPT; \
 iptables -I OUTPUT -o wg0 -j ACCEPT; \
 iptables -A INPUT -j DROP; \
 iptables -A FORWARD -j DROP; \
 iptables -A OUTPUT -j DROP

PostDown = iptables -D INPUT -i wg0 -j ACCEPT; \
 iptables -D FORWARD -i wg0 -j ACCEPT; \
 iptables -D OUTPUT -o wg0 -j ACCEPT; \
 iptables -D INPUT -j DROP; \
 iptables -D FORWARD -j DROP; \
 iptables -D OUTPUT -j DROP
```

The script blocks all inbound and outbound traffic except traffic through the WireGuard interface. When the tunnel disconnects, your system becomes effectively isolated, preventing any unencrypted traffic from leaking.

For IPv6 compatibility, add parallel ip6tables rules:

```ini
PostUp = ip6tables -I INPUT -i wg0 -j ACCEPT; \
 ip6tables -I FORWARD -i wg0 -j ACCEPT; \
 ip6tables -A INPUT -j DROP; \
 ip6tables -A FORWARD -j DROP

PostDown = ip6tables -D INPUT -i wg0 -j ACCEPT; \
 ip6tables -D FORWARD -i wg0 -j ACCEPT; \
 ip6tables -D INPUT -j DROP; \
 ip6tables -D FORWARD -j DROP
```

## Custom DNS Resolution

Sometimes you need specific DNS servers for VPN-connected hosts. PostUp lets you update your resolv.conf or configure systemd-resolved:

```ini
[Interface]
PrivateKey = <key>
Address = 10.2.0.5/24
DNS = 10.0.0.53

PostUp = mkdir -p /etc/netns/wg0; \
 echo "nameserver 1.1.1.1" > /etc/netns/wg0/resolv.conf; \
 echo "nameserver 1.0.0.1" >> /etc/netns/wg0/resolv.conf; \
 resolvconf -a wg0 -m 0 -x

PostDown = resolvconf -d wg0 -f
```

This creates a network namespace specifically for the VPN interface, ensuring DNS queries from the tunnel use your preferred servers regardless of system-wide DNS settings.

## Managing Multiple Peers

Advanced setups often involve multiple peers with different routing requirements. Use interface-specific PostUp scripts:

```ini
[Peer]
PublicKey = <peer1-key>
Endpoint = us-east.vpn.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
PostUp = ip route add default dev wg0 via 10.2.0.1; \
 iptables -t mangle -A PREROUTING -s 192.168.1.100 -j MARK --set-mark 1

[Peer]
PublicKey = <peer2-key>
Endpoint = eu.vpn.com:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

Each peer can have its own routing logic, allowing you to route traffic to different VPN endpoints based on source IP, destination, or other criteria.

## Debugging Your Scripts

When scripts don't work as expected, debugging is essential. Add logging to your scripts:

```bash
#!/bin/bash
LOGFILE="/var/log/wireguard-debug.log"
echo "$(date): PostUp executing - $@" >> "$LOGFILE"

# Your commands here
ip route add default dev wg0 2>> "$LOGFILE"

echo "$(date): PostUp complete" >> "$LOGFILE"
```

Also verify your WireGuard logs:

```bash
journalctl -u wg-quick@wg0 -f
```

Common issues include permission problems (run scripts as root), timing issues (add small delays with `sleep`), and interface name mismatches.

## Security Considerations

PostUp and PostDown scripts run with root privileges, making security critical:

- Store sensitive scripts in root-owned directories with restricted permissions (700)
- Validate all variables before use to prevent injection attacks
- Avoid storing credentials in scripts—use WireGuard's built-in key management
- Test scripts in a non-production environment before deployment

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does WireGuard offer a free tier?**

Most major tools offer some form of free tier or trial period. Check WireGuard's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [WireGuard Postup Postdown Scripts For Advanced Routing](/privacy-tools-guide/wireguard-postup-postdown-scripts-for-advanced-routing-configuration/)
- [WireGuard Performance Tuning for Large File Transfer](/privacy-tools-guide/wireguard-performance-tuning-large-file-transfer-optimizatio/)
- [Set Up a Personal VPN with WireGuard](/privacy-tools-guide/wireguard-personal-vpn-setup-guide)
- [WireGuard Dynamic Endpoint Update](/privacy-tools-guide/wireguard-dynamic-endpoint-update-how-roaming-between-networks-works/)
- [Wireguard Android Battery Optimization Settings](/privacy-tools-guide/wireguard-android-battery-optimization-settings-without-brea/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
