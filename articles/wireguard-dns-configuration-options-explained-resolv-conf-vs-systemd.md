---
layout: default
title: "WireGuard DNS Configuration Options Explained"
description: "Complete guide to WireGuard DNS configuration. Learn how to set DNS using resolv.conf, systemd-resolved, and wg-quick. Compare methods for Linux, Android, and"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /wireguard-dns-configuration-options-explained-resolv-conf-vs-systemd/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, comparison]
---

{% raw %}

Configuring DNS properly with WireGuard is essential for privacy and functionality. When your VPN tunnel routes all traffic through WireGuard, you need to ensure DNS queries also go through the tunnel—otherwise your DNS requests could leak outside the encrypted connection, undermining your privacy. This guide covers the primary DNS configuration methods, comparing the traditional `/etc/resolv.conf` approach with the more modern `systemd-resolved` integration.

## Why DNS Configuration Matters with WireGuard

WireGuard operates at the kernel level, handling encrypted tunnel traffic efficiently. However, DNS resolution happens separately from the VPN tunnel itself. By default, your system continues using the DNS servers configured by your ISP or network manager. This creates a DNS leak—your traffic may go through the VPN tunnel, but your DNS queries remain visible to your ISP and potentially other observers.

Configuring DNS to work with WireGuard ensures that all your DNS queries also travel through the encrypted tunnel. This prevents DNS leaks and maintains consistent privacy regardless of which applications you're using. The method you choose depends on your Linux distribution, network setup, and whether you're running systemd.

## Method 1: Using wg-quick with the DNS Directive

The simplest approach for most users is using WireGuard's built-in `wg-quick` tool, which handles DNS configuration automatically through the tunnel interface settings. This method works on most Linux distributions and is the recommended approach for desktop and laptop configurations.

In your WireGuard configuration file (typically stored in `/etc/wireguard/wg0.conf`), add the `DNS` directive under the `[Interface]` section:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

When you bring up the tunnel with `wg-quick up wg0`, WireGuard automatically configures the specified DNS server. You can specify multiple DNS servers by listing them space-separated:

```ini
DNS = 1.1.1.1 1.0.0.1
```

This method handles DNS configuration at the interface level, making it work system-wide once the tunnel is active. The DNS setting persists only while the tunnel is up, which is the desired behavior for most use cases.

## Method 2: Direct /etc/resolv.conf Configuration

The traditional method involves directly editing `/etc/resolv.conf`, which is the primary configuration file for DNS resolvers on most Unix-like systems. This file contains nameserver entries that define which DNS servers your system uses.

Before modifying resolv.conf, you should understand how your system manages this file. Many distributions use resolvconf or similar tools that can overwrite manual changes. On systems without systemd-resolved, you might need to protect your changes.

To configure DNS manually, edit the resolv.conf file:

```bash
sudo nano /etc/resolv.conf
```

Add your preferred DNS servers:

```conf
nameserver 1.1.1.1
nameserver 1.0.0.1
```

For privacy-focused setups, you might use:

```conf
nameserver 9.9.9.9
nameserver 149.112.112.112
```

The main advantage of this method is transparency—you know exactly which DNS servers are being used. However, this approach has drawbacks: network manager services might overwrite your changes, the configuration doesn't automatically revert when the VPN disconnects, and you need to manually manage the DNS servers when switching between VPN and non-VPN states.

Some users create scripts to manage resolv.conf changes:

```bash
#!/bin/bash
# Save as /usr/local/bin/vpn-dns-up
cp /etc/resolv.conf /etc/resolv.conf.backup
echo "nameserver 1.1.1.1" > /etc/resolv.conf
echo "nameserver 1.0.0.1" >> /etc/resolv.conf

# Save as /usr/local/bin/vpn-dns-down
cp /etc/resolv.conf.backup /etc/resolv.conf
```

## Method 3: systemd-resolved Integration

Modern Linux distributions using systemd include systemd-resolved, which manages DNS configuration through a local stub resolver. This approach integrates cleanly with NetworkManager and provides benefits like DNS caching and split-tunneling support.

Systemd-resolved maintains its own DNS configuration stored in `/run/systemd/resolve/resolv.conf` or `/run/systemd/resolve/stub-resolv.conf`. To configure DNS for WireGuard with systemd-resolved, you have several options.

The first approach uses wg-quick's built-in systemd-resolved support. Modern versions of wg-quick automatically interact with systemd-resolved when available. Add the DNS directive in your wg0.conf as shown earlier, and wg-quick will notify systemd-resolved of the DNS changes.

Alternatively, you can configure DNS manually through systemd-resolved:

```bash
# View current DNS configuration
resolvectl status

# Set DNS for a specific interface
sudo resolvectl dns wg0 1.1.1.1 1.0.0.1

# Verify the configuration
resolvectl status wg0
```

systemd-resolved supports domain-specific DNS routing, which is useful for split-tunnel configurations. You can route DNS for specific domains through different servers:

```bash
# Route corporate domain through a different DNS when on VPN
sudo resolvectl domain wg0 corporate.example.com ~internal.corp
```

This flexibility makes systemd-resolved the preferred method for complex network configurations. The stub resolver also provides DNS caching, which can slightly improve response times for repeated queries.

## Comparing resolv.conf and systemd-resolved

The choice between these methods depends on your system configuration and requirements. Here's a practical comparison:

**resolv.conf method advantages:**
- Works on all Unix-like systems, including non-systemd distributions
- Transparent and easy to understand
- No dependency on systemd services
- Works consistently across different network setups

**resolv.conf method disadvantages:**
- Can be overwritten by network managers
- Requires manual management of VPN up/down scripts
- No built-in DNS caching
- May not persist across network changes

**systemd-resolved advantages:**
- Automatic integration with NetworkManager
- DNS caching improves performance
- Supports domain-specific routing
- Clean up/down handling with wg-quick
- Modern and actively maintained

**systemd-resolved disadvantages:**
- Requires systemd-based distribution
- Slightly more complex to debug
- Stub resolver adds a small overhead
- May conflict with custom resolv.conf configurations

For most desktop Linux users with modern distributions (Ubuntu 22.04+, Fedora, Arch), systemd-resolved with wg-quick provides the smoothest experience. For servers, embedded systems, or non-systemd distributions, the direct resolv.conf approach offers more control.

## DNS Configuration for Mobile and Router Setups

WireGuard runs on various platforms beyond Linux desktops, and DNS configuration varies accordingly.

On Android, WireGuard app handles DNS automatically when you configure it in the tunnel settings. The app includes a DNS field in the interface configuration that works without any system-level changes. IOS users similarly configure DNS within the WireGuard app interface.

For router implementations (OpenWrt,pfSense, or dedicated hardware), DNS configuration typically happens at the network level. On OpenWrt with wg-quick, add the DNS directive to your WireGuard interface configuration in `/etc/config/network`:

```bash
config interface 'wg0'
 option proto 'wireguard'
 option privatekey '<your-key>'
 list addresses '10.0.0.2/32'
 list dns '1.1.1.1'
```

This ensures all devices behind the router use the VPN's DNS servers, providing leak protection for your entire network.

## Troubleshooting DNS Leaks

After configuring DNS with WireGuard, verify that your DNS requests aren't leaking:

```bash
# Check which DNS server is currently in use
cat /etc/resolv.conf

# Test DNS resolution and see which server responds
dig example.com

# Use an online DNS leak test
# Visit https://dnsleaktest.com or similar service
```

If you're using systemd-resolved:

```bash
# Check which DNS servers are active
resolvectl status

# Look for your WireGuard interface
resolvectl status wg0
```

Common issues include NetworkManager overriding resolv.conf (solve by configuring NetworkManager to ignore VPN DNS), wg-quick not supporting your systemd version (update wg-quick or use manual configuration), and stub resolver conflicts (check `/etc/resolv.conf` points to 127.0.0.53).

## Best Practices for WireGuard DNS

When configuring DNS for WireGuard, consider these recommendations:

Use encrypted DNS servers when possible. Cloudflare (1.1.1.1), Quad9 (9.9.9.9), and Google (8.8.8.8) all support DNS-over-TLS and DNS-over-HTTPS, providing additional privacy layers beyond the VPN tunnel.

Configure a primary and secondary DNS server. If your primary DNS becomes unavailable, the secondary ensures continued connectivity:

```ini
DNS = 1.1.1.1 9.9.9.9
```

Test your configuration after setup. DNS leaks can occur due to application-level DNS overrides or VPN reconnection issues. Regular testing ensures your privacy remains protected.

Consider DNS caching implications. If you're using systemd-resolved, be aware that cached entries persist after tunnel teardown. Flush the cache if you switch DNS configurations frequently:

```bash
sudo resolvectl flush-caches
```

Proper DNS configuration completes your WireGuard privacy setup. Whether you choose the simplicity of wg-quick's built-in handling or the flexibility of manual systemd-resolved configuration, ensuring DNS queries travel through your VPN tunnel prevents one of the most common privacy leaks in VPN setups.



## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Does WireGuard offer a free tier?**

Most major tools offer some form of free tier or trial period. Check WireGuard's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.


**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


## Related Articles

- [Wireguard Postup Postdown Scripts For Advanced Routing.](/privacy-tools-guide/wireguard-postup-postdown-scripts-for-advanced-routing-configuration/)
- [Openvpn Push Route Configuration Selective Routing Explained](/privacy-tools-guide/openvpn-push-route-configuration-selective-routing-explained-step-by-step/)
- [WireGuard Persistent Keepalive Setting Explained](/privacy-tools-guide/wireguard-persistent-keepalive-setting-explained-when-to-enable-it/)
- [Anonymous Vehicle Registration Options For Keeping Home Addr](/privacy-tools-guide/anonymous-vehicle-registration-options-for-keeping-home-addr/)
- [Hardened Firefox Privacy Configuration Guide](/privacy-tools-guide/hardened-firefox-privacy-configuration/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
