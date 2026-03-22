---
layout: default
title: "How to Set Up Mullvad VPN on Linux"
description: "Install and configure Mullvad VPN on Linux using the official app and WireGuard CLI, set up the kill switch, enable split tunneling, and verify no leaks"
date: 2026-03-22
author: theluckystrike
permalink: /mullvad-vpn-linux-setup-guide/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]

{% raw %}

# How to Set Up Mullvad VPN on Linux

Mullvad is notable for accepting cash and Monero payments, requiring no email to sign up (only an account number), and publishing audit results for their no-logs policy. This guide covers the official Linux app, manual WireGuard setup, kill switch configuration, and leak testing.
---

## Key Takeaways

- **Write down your 16-digit account number**: this is your only credential
3.
- **For maximum anonymity**: buy with Monero or mail cash

No email, no username, no phone number.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Consider a security review**: if your application handles sensitive user data.

## Account Setup

Mullvad doesn't require an email address:

1. Go to `mullvad.net` → Generate account number
2. Write down your 16-digit account number — this is your only credential
3. Add time: purchase with card, PayPal, Bitcoin, Monero, or cash
4. For maximum anonymity: buy with Monero or mail cash

No email, no username, no phone number. If you lose the account number, there's no recovery — keep it in your password manager.

---

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install the Mullvad App

**Debian/Ubuntu:**

```bash
# Add Mullvad repository
sudo curl -fsSLo /usr/share/keyrings/mullvad-keyring.asc \
  https://repository.mullvad.net/deb/mullvad-keyring.asc

echo "deb [signed-by=/usr/share/keyrings/mullvad-keyring.asc arch=$( dpkg --print-architecture )] \
  https://repository.mullvad.net/deb/stable $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/mullvad.list

sudo apt update && sudo apt install mullvad-vpn
```

**Fedora/RHEL:**

```bash
sudo dnf config-manager --add-repo \
  https://repository.mullvad.net/rpm/stable/mullvad.repo
sudo dnf install mullvad-vpn
```

**Arch Linux:**

```bash
yay -S mullvad-vpn-bin
```

---

### Step 2: CLI Setup and Connect

Mullvad's CLI tool is `mullvad`:

```bash
# Log in with your account number
mullvad account login 1234567890123456

# Verify account status
mullvad account get
# Shows: time remaining, account number

# Connect to VPN (auto-selects nearest server)
mullvad connect

# Check connection status
mullvad status
# Should show: Connected to [server] via WireGuard

# Check your public IP (should now be Mullvad's IP)
curl -s https://am.i.mullvad.net/json | python3 -m json.tool
```

---

### Step 3: Server Selection

```bash
# List available servers
mullvad relay list

# Filter by country
mullvad relay list | grep "us "

# Set preferred location
mullvad relay set location se    # Sweden
mullvad relay set location us nyc  # New York City

# Set specific protocol
mullvad relay set tunnel-protocol wireguard

# Use a specific server
mullvad relay set hostname se-got-wg-001

# Reconnect with new settings
mullvad reconnect
```

---

### Step 4: Enable the Kill Switch

The kill switch blocks all internet traffic if the VPN connection drops:

```bash
# Enable kill switch
mullvad lockdown-mode set on

# Verify status
mullvad lockdown-mode get
# Shows: Lockdown mode enabled

# Test: disconnect and verify internet is blocked
mullvad disconnect
ping 8.8.8.8   # Should fail
curl https://ifconfig.me  # Should fail
mullvad connect          # Reconnect to restore access
```

**How it works**: Mullvad's kill switch uses network rules (via `nftables` or `iptables`) that allow only traffic through the WireGuard tunnel interface. All other outbound traffic is dropped.

---

### Step 5: DNS Configuration

```bash
# View current DNS servers (Mullvad's by default)
mullvad dns get

# Use Mullvad's ad/tracker-blocking DNS
mullvad dns set default --block-ads --block-trackers

# Use a custom DNS server (e.g., your local Pi-hole)
mullvad dns set custom 192.168.1.10

# Verify DNS isn't leaking
curl -s https://am.i.mullvad.net/json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('mullvad_exit_ip', False), d.get('mullvad_server', 'none'))"
```

---

### Step 6: Split Tunneling

Split tunneling sends specific apps outside the VPN tunnel while others go through it:

```bash
# Enable split tunneling
mullvad split-tunnel set enable

# Add an app to bypass the VPN
mullvad split-tunnel app add /usr/bin/steam
mullvad split-tunnel app add /usr/bin/firefox

# List bypassed apps
mullvad split-tunnel app list

# Remove from bypass list
mullvad split-tunnel app remove /usr/bin/firefox

# To route only specific apps THROUGH the VPN (inverse mode)
# Not natively supported in Mullvad — requires custom routing
```

---

### Step 7: Manual WireGuard Setup (Without the App)

For servers, minimal installs, or when you prefer to manage the connection directly:

```bash
# Install WireGuard
sudo apt install wireguard

# Generate your device key in Mullvad account
# mullvad.net → Account → WireGuard keys → Generate
# Download the config file for your chosen server, or:

# Use the Mullvad API to get server configs
curl -s https://api.mullvad.net/www/relays/wireguard/ | \
  python3 -c "import json,sys; [print(r['hostname'], r['ipv4_addr_in']) for r in json.load(sys.stdin)['wireguard'] if 'se-got' in r['hostname']]"
```

```ini
# /etc/wireguard/mullvad-se.conf
[Interface]
PrivateKey = YOUR_PRIVATE_KEY_FROM_MULLVAD_ACCOUNT
Address = 10.68.x.x/32
DNS = 10.64.0.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = 185.x.x.x:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
# Connect
sudo wg-quick up mullvad-se

# Check status
sudo wg show

# Set as service
sudo systemctl enable wg-quick@mullvad-se

# Disconnect
sudo wg-quick down mullvad-se
```

---

### Step 8: DAITA (Defense Against AI-Guided Traffic Analysis)

Mullvad's DAITA feature adds random padding and artificial traffic to your connection to frustrate machine learning-based traffic fingerprinting:

```bash
# Enable DAITA (CLI)
mullvad tunnel wireguard set daita enable

# Check status
mullvad tunnel wireguard get | grep -i daita
```

DAITA adds bandwidth overhead (roughly 15-30%) but provides meaningful protection against traffic analysis attacks.

---

### Step 9: Verify No Leaks

```bash
# Full leak test
curl -s https://am.i.mullvad.net/json | python3 -m json.tool
# Check: mullvad_exit_ip: true, mullvad_server: [server]

# DNS leak test
for i in {1..5}; do
  nslookup whoami.akamai.net
done
# All responses should come from Mullvad's DNS or your configured DNS

# IPv6 leak test
curl -6 https://ipv6.icanhazip.com 2>/dev/null || echo "IPv6 not routed (good)"
# If you get an IPv6 address, check Mullvad's IPv6 settings

# WebRTC leak (browser-based)
# Open: browserleaks.com/webrtc
# Your real IP should NOT appear

# Disable IPv6 if leaking
mullvad tunnel wireguard set ipv6 disable
```

---

### Step 10: Autostart on Boot

```bash
# Enable Mullvad to connect automatically at startup
mullvad auto-connect set on

# Verify
mullvad auto-connect get
# Shows: Autoconnect: on

# For systemd service (manual WireGuard):
sudo systemctl enable wg-quick@mullvad-se
```

---

### Step 11: Multihop (Double VPN) Configuration

Mullvad supports multihop routing, where your traffic enters one server and exits through a different server in another country. This adds a layer of separation — the entry server knows your IP but not your destination, while the exit server knows your destination but not your real IP.

```bash
# Enable multihop
mullvad relay set tunnel-protocol wireguard
mullvad relay set wireguard --use-multihop on

# Set entry and exit locations separately
mullvad relay set entry location se      # Entry: Sweden
mullvad relay set location us nyc        # Exit: New York

# Verify multihop is active
mullvad status
# Should show: Connected to [exit-server] via [entry-server]
```

Multihop increases latency by 20–60ms depending on geographic distance between entry and exit nodes. Use it when you need to obscure which VPN provider's exit IP you're using, or when operating in environments where known Mullvad IP ranges are monitored.

---

### Step 12: Configure Mullvad with a Firewall (nftables)

On headless servers running Mullvad's manual WireGuard setup, you may want explicit firewall rules rather than relying solely on the `wg-quick` AllowedIPs mechanism. The following nftables configuration enforces that all outbound traffic routes through the WireGuard interface:

```bash
# /etc/nftables-mullvad.conf
table inet mullvad_firewall {
    chain output {
        type filter hook output priority 0; policy drop;

        # Allow traffic on WireGuard interface
        oifname "mullvad-se" accept

        # Allow loopback
        oifname "lo" accept

        # Allow DHCP (to maintain LAN connectivity)
        udp dport 67 accept

        # Allow WireGuard handshake to the endpoint
        ip daddr 185.0.0.0/8 udp dport 51820 accept
    }
}
```

Apply with:

```bash
sudo nft -f /etc/nftables-mullvad.conf
```

This kill switch approach operates at the firewall layer independently of the VPN client, meaning it survives application crashes and provides protection before the VPN process starts on boot.

---

## Troubleshooting Common Linux Issues

**Problem: DNS leaks despite VPN being connected**

Check whether your system is using `systemd-resolved` and whether it respects the DNS pushed by WireGuard:

```bash
resolvectl status
# Look for: DNS Servers line on the WireGuard interface

# If wrong DNS is showing, force it:
sudo resolvectl dns mullvad-se 10.64.0.1
sudo resolvectl domain mullvad-se "~."
```

**Problem: VPN connects but traffic still routes through physical interface**

Verify routing table priorities:

```bash
ip route show table main | grep default
# Should point to the WireGuard interface when connected

# If not, check AllowedIPs in your config:
sudo wg show
# AllowedIPs should include 0.0.0.0/0 for full tunnel routing
```

**Problem: High latency or packet loss on WireGuard tunnel**

WireGuard is sensitive to MTU mismatches across different network paths. Try reducing the MTU on your WireGuard interface:

```bash
# In /etc/wireguard/mullvad-se.conf, add to [Interface]:
MTU = 1380

# Then restart:
sudo wg-quick down mullvad-se && sudo wg-quick up mullvad-se
```

A value of 1380 works reliably across most networks, including those with PPPoE encapsulation that reduce effective MTU below the standard 1500.

---

## Related Reading

- [Proton VPN vs Mullvad Speed Test Privacy Audit 2026](/privacy-tools-guide/proton-vpn-vs-mullvad-speed-test-privacy-audit-2026/)
- [OpenWrt VPN Router WireGuard Setup](/privacy-tools-guide/openwrt-vpn-router-wireguard-setup/)
- [How to Set Up a VPN on Your Router](/privacy-tools-guide/vpn-on-router-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to set up mullvad vpn on linux?**

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
