---
layout: default
title: "How to Set Up Mullvad VPN on Linux"
description: "Install and configure Mullvad VPN on Linux using the official app and WireGuard CLI, set up the kill switch, enable split tunneling, and verify no leaks"
date: 2026-03-22
author: theluckystrike
permalink: /mullvad-vpn-linux-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

# How to Set Up Mullvad VPN on Linux

Mullvad is notable for accepting cash and Monero payments, requiring no email to sign up (only an account number), and publishing audit results for their no-logs policy. This guide covers the official Linux app, manual WireGuard setup, kill switch configuration, and leak testing.
---

## Account Setup

Mullvad doesn't require an email address:

1. Go to `mullvad.net` → Generate account number
2. Write down your 16-digit account number — this is your only credential
3. Add time: purchase with card, PayPal, Bitcoin, Monero, or cash
4. For maximum anonymity: buy with Monero or mail cash

No email, no username, no phone number. If you lose the account number, there's no recovery — keep it in your password manager.

---

## Install the Mullvad App

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

## CLI Setup and Connect

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

## Server Selection

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

## Enable the Kill Switch

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

## DNS Configuration

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

## Split Tunneling

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

## Manual WireGuard Setup (Without the App)

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

## DAITA (Defense Against AI-Guided Traffic Analysis)

Mullvad's DAITA feature adds random padding and artificial traffic to your connection to frustrate machine learning-based traffic fingerprinting:

```bash
# Enable DAITA (CLI)
mullvad tunnel wireguard set daita enable

# Check status
mullvad tunnel wireguard get | grep -i daita
```

DAITA adds bandwidth overhead (roughly 15-30%) but provides meaningful protection against traffic analysis attacks.

---

## Verify No Leaks

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

## Autostart on Boot

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

