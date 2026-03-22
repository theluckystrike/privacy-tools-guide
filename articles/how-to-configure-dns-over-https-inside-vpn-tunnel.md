---
layout: default
title: "How to Configure DNS over HTTPS Inside a VPN"
description: "Route DNS queries through HTTPS inside your VPN tunnel on Windows, macOS, and Linux. Prevent DNS leaks with Cloudflare or NextDNS configs."
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-configure-dns-over-https-inside-vpn-tunnel/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]
---

{% raw %}

When you use a VPN, your internet traffic is encrypted and routed through the VPN server, hiding your browsing activity from your ISP. However, even with a VPN, your DNS requests can still leak information about your browsing habits. DNS over HTTPS (DoH) adds an additional layer of privacy by encrypting your DNS queries, making it impossible for anyone—including your VPN provider—to see what domains you're accessing.

In this guide, we'll explore how to configure DNS over HTTPS inside a VPN tunnel, why it matters, and the various methods to implement it across different platforms and VPN protocols.

## Understanding DNS Leaks and Why DoH Inside VPN Matters

### What is a DNS Leak?

Every time you visit a website, your computer needs to translate the human-readable domain name (like example.com) into an IP address. This process is called DNS (Domain Name System) resolution. By default, your computer sends these DNS queries to your ISP's DNS servers, which can log every website you visit.

When you connect to a VPN, ideally all your traffic—including DNS queries—should go through the encrypted VPN tunnel. However, due to misconfigurations or IPv6 compatibility issues, DNS queries can sometimes "leak" outside the VPN tunnel, exposing your browsing activity to your ISP or other observers.

### Why Combine DoH with VPN?

DNS over HTTPS encrypts your DNS queries using HTTPS protocol, making them indistinguishable from regular web traffic. When you combine DoH with a VPN:

1. **Double Encryption**: Your DNS queries are encrypted twice—once by the VPN tunnel and again by HTTPS
2. **ISP Privacy**: Your ISP cannot see even the domains you're accessing
3. **VPN Provider Privacy**: Your VPN provider cannot log your DNS queries
4. **Bypass Local DNS Blocks**: DoH can bypass local DNS-based content filtering
5. **Protection Against DNS Spoofing**: Encrypted DNS is resistant to man-in-the-middle DNS attacks

## Configuring DoH Inside Different VPN Setups

### Method 1: WireGuard with DoH Stubby Configuration

WireGuard is known for its simplicity and performance. Here's how to configure DoH with WireGuard on Linux:

#### Step 1: Install Required Packages

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install stubby dnsmasq

# Fedora/RHEL
sudo dnf install stubby dnsmasq
```

#### Step 2: Configure Stubby for DNS over HTTPS

Edit the Stubby configuration file:

```bash
sudo nano /etc/stubby/stubby.yml
```

Add the following configuration:

```yaml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_HTTPS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
edns_client_subnet_private: 0
round_robin_upstreams: 1
listen_addresses:
  - 127.0.0.1@53530
upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_auth_name: "cloudflare-dns.com"
    tls_pubkey_pinset:
      - digest: "SHA256"
        value: y/oE5kfNgrX5qYJqJVq/3L2n6WcP8jR5vN3kGmT9sM=
  - address_data: 8.8.8.8
    tls_auth_name: "dns.google"
    tls_pubkey_pinset:
      - digest: "SHA256"
        value: JSMUlqH2g/3BE/h2+aSxL+jEv9IU/A9zZ5Xu2HI7RoM=
```

#### Step 3: Configure WireGuard to Use Local DNS

Edit your WireGuard configuration:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Add or modify the DNS setting:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/24
DNS = 127.0.0.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

#### Step 4: Configure Dnsmasq as Local Caching Resolver

```bash
sudo nano /etc/dnsmasq.conf
```

Add these lines:

```conf
no-resolv
server=127.0.0.1#53530
cache-size=1000
```

#### Step 5: Restart Services

```bash
sudo systemctl restart stubby
sudo systemctl restart dnsmasq
sudo wg-quick down wg0
sudo wg-quick up wg0
```

Now all your DNS queries will go through DoH before being sent through the WireGuard VPN tunnel.

### Method 2: OpenVPN with DoH Configuration

OpenVPN doesn't natively support DoH, but you can chain a local DoH resolver similar to the WireGuard method.

#### Using dnscrypt-proxy with OpenVPN

```bash
# Install dnscrypt-proxy
sudo apt install dnscrypt-proxy

# Configure dnscrypt-proxy
sudo nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

Edit the configuration:

```toml
listen_addresses = ['127.0.0.1:53']
max_clients = 250
ipv4_servers = true
ipv6_servers = false
dnscrypt_servers = true
doh_servers = true
require_dnssec = true
require_nolog = true
disable_ipv6 = true
cache = true
cache_size = 10000
cache_min_ttl = 600
cache_max_ttl = 86400
cache_neg_ttl = 3600
```

Configure OpenVPN to use this DNS:

```ini
# Add to your OpenVPN client config
script-security 2
up /etc/openvpn/update-dns
down /etc/openvpn/update-dns
```

Create the update script:

```bash
#!/bin/bash
# /etc/openvpn/update-dns
if [ "$1" = "up" ]; then
    # When VPN comes up, use local DoH resolver
    echo "nameserver 127.0.0.1" > /etc/resolv.conf
    chattr +i /etc/resolv.conf
elif [ "$1" = "down" ]; then
    # When VPN goes down, restore original DNS
    chattr -i /etc/resolv.conf
    cp /etc/resolv.conf.vpn /etc/resolv.conf
fi
```

### Method 3: Configuring DoH on Windows with VPN

Windows 11 has native DoH support, which you can configure to work with your VPN:

#### Step 1: Enable DoH in Windows Settings

1. Open Settings → Network & Internet → Wi-Fi or Ethernet
2. Click on your active network connection
3. Scroll to "DNS server assignment"
4. Select "Manual"
5. Enable "IPv4"
6. Enter a DoH-compatible DNS server:
 - Cloudflare: `1.1.1.1`
 - Google: `8.8.8.8`
 - Quad9: `9.9.9.9`
7. Set "Preferred DNS" and "Alternate DNS" to the same IP
8. Under "Preferred DNS encryption", select "Encrypted (DNS over HTTPS)"
9. Repeat for IPv6 if desired

#### Step 2: Configure VPN to Use System DNS

When your VPN connects, it may override your DNS settings. To ensure your VPN uses the DoH-enabled system DNS:

1. Open your VPN client's settings
2. Look for "DNS Settings" or "Network Settings"
3. Disable "Use VPN provider's DNS" or similar option
4. Select "Use system DNS" or "Automatic"

### Method 4: macOS DoH with VPN Tunnel

On macOS, you can use the built-in DoH support or third-party applications:

#### Using macOS System Preferences

```bash
# For macOS Ventura and later
# Enable DoH via Terminal
sudo networksetup -setdnsservers "Wi-Fi" 1.1.1.1 8.8.8.8

# Note: Native DoH GUI settings are in System Preferences > Network > Wi-Fi > Advanced > DNS
```

#### Using dnscrypt-proxy with macOS VPN

```bash
# Install Homebrew if not installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install dnscrypt-proxy
brew install dnscrypt-proxy

# Copy and edit configuration
cp /usr/local/etc/dnscrypt-proxy/dnscrypt-proxy.toml.sample /usr/local/etc/dnscrypt-proxy/dnscrypt-proxy.toml

# Edit to use DoH servers
nano /usr/local/etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

Configure your VPN client to use `127.0.0.1` as the DNS server.

### Method 5: Router-Level Configuration

Configuring DoH at the router level protects all devices on your network:

#### Using OpenWrt with DoH

1. Install required packages:
```bash
opkg update
opkg install https-dns-proxy
```

2. Configure https-dns-proxy:
```bash
# Edit /etc/config/https-dns-proxy
config https-dns-proxy
    option interpreter '/usr/bin/https-dns-proxy'
    option bootstrap_dns '1.1.1.1,8.8.8.8'
    option resolver_url 'https://cloudflare-dns.com/dns-query'
    option listen_addr '127.0.0.1'
    option listen_port '5053'
```

3. Add firewall rule to redirect DNS:
```bash
config redirect
    option name 'Redirect-DNS'
    option src 'lan'
    option proto 'udp'
    option src_port '53'
    option dest_port '5053'
    option target 'REDIRECT'
```

## Verifying Your DoH Inside VPN Configuration

### Test for DNS Leaks

Use these online tools to verify your configuration:

1. **dnsleaktest.com**: Run the extended test to check which DNS servers are being used
2. **ipleak.net**: Check for IP and DNS leaks
3. **browserleaks.com**: privacy tests

### Verify DoH is Working

To verify DoH is actually encrypting your DNS:

```bash
# On Linux, use tcpdump to monitor DNS traffic
sudo tcpdump -i any -n port 53 or port 443

# You should NOT see plaintext DNS queries on port 53
# You should see encrypted HTTPS traffic on port 443
```

### Check Which DNS Resolution You're Using

```bash
# Test with dig
dig +short whoami.cloudflare.com @1.1.1.1

# Check your DNS resolver
nslookup example.com
```

## Troubleshooting Common Issues

### DNS Resolution Fails After Enabling DoH

- Check that your DoH provider is reachable: `curl -v https://1.1.1.1/dns-query`
- Verify firewall rules allow HTTPS outbound
- Check that stubby/dnscrypt-proxy is running: `systemctl status stubby`

### VPN Connection Drops

- Ensure "PersistentKeepalive" is set in your VPN config
- Check that your DoH resolver has a fallback
- Verify network connectivity

### DNS Conflicts with VPN Provider

- Some VPNs force their own DNS servers
- Use a local DNS forwarder (dnsmasq) to override
- Configure your VPN client to use "Use system DNS" option

## Advanced: Split DNS with DoH

For more granular control, implement split DNS:

```bash
# Example: Use DoH for specific domains only
# Configure in /etc/dnsmasq.conf
server=/corporate-internal.com/1.1.1.1
server=/private-network.local/1.1.1.1
bogus-priv
```

## Frequently Asked Questions

**How long does it take to configure dns over https inside a vpn?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}

## Related Articles

- [How to Verify Your VPN is Not Leaking DNS Requests in 2026](/privacy-tools-guide/how-to-verify-your-vpn-is-not-leaking-dns-requests/)
- [How to Configure DNS Over HTTPS (DoH) for Privacy in 2026](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/---)
- [How To Tell If Your Dns Has Been Hijacked Symptoms](/privacy-tools-guide/how-to-tell-if-your-dns-has-been-hijacked-symptoms-check/)
- [How to Verify VPN Is Working Correctly 2026](/privacy-tools-guide/how-to-verify-vpn-is-working-correctly-2026/)
- [How to Set Up Encrypted DNS on All Devices 2026](/privacy-tools-guide/how-to-set-up-encrypted-dns-on-all-devices-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
