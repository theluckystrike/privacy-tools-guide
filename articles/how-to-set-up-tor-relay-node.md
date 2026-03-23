---
layout: default
title: "How to Set Up a Tor Relay"
description: "Step-by-step guide to configuring a Tor middle relay on Linux, including torrc settings, bandwidth limits, firewall rules, and monitoring your relay"
date: 2026-03-21
author: theluckystrike
permalink: /how-to-set-up-tor-relay-node/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Running a Tor relay contributes bandwidth to the Tor network and helps more people access the internet privately. This guide covers setting up a **middle relay** — the most appropriate choice for most people. Middle relays carry encrypted traffic between other Tor nodes; they never see plaintext traffic and have a much lower legal risk profile than exit relays.

Exit relays (where traffic exits onto the public internet) carry more legal complexity and are outside the scope of this guide. If you want to run an exit relay, read the Tor Project's legal guidance first.

## Prerequisites

- A Linux server with a static IP address (VPS, dedicated server, or a home server with a stable connection)
- At least 10 Mbit/s of available bandwidth (uplink and downlink)
- A public IPv4 or IPv6 address
- Debian, Ubuntu, or a compatible distribution (this guide uses Debian/Ubuntu)

## Step 1: Install Tor

Use the Tor Project's official repository, not the version in your distro's default repos (which may be outdated):

```bash
# Install dependencies
sudo apt update && sudo apt install -y curl gpg

# Add Tor Project repository
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] \
  https://deb.torproject.org/torproject.org $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/tor.list

# Import signing key
curl -s https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/tor-archive-keyring.gpg >/dev/null

# Install
sudo apt update && sudo apt install -y tor deb.torproject.org-keyring
```

## Step 2: Configure the Relay

Edit the Tor configuration file:

```bash
sudo nano /etc/tor/torrc
```

Replace the contents with this relay configuration (adjust values marked with `##`):

```
## Relay nickname (no spaces, alphanumeric only, max 19 chars)
Nickname MyPrivacyRelay

## Contact info shown in relay directory (optional but good practice)
ContactInfo your@email.example

## OR port - the port Tor uses to receive relay connections
ORPort 9001

## Directory port - serves relay consensus data to clients
DirPort 9030

## Operating as a relay (not exit, not bridge)
ExitPolicy reject *:*

## Bandwidth settings - adjust to match your available capacity
## 100 MBytes per second sustained
RelayBandwidthRate 10 MBytes
## Allow up to 20 MBytes/s burst
RelayBandwidthBurst 20 MBytes
## Optional: limit total monthly traffic (example: 500 GB/month)
# AccountingMax 500 GBytes
# AccountingStart month 1 00:00

## IPv6 support (if your server has IPv6)
# ORPort [2001:db8::1]:9001

## Log settings
Log notice file /var/log/tor/notices.log
```

Save and close the file.

## Step 3: Configure the Firewall

Allow traffic on the ORPort and DirPort:

```bash
# UFW
sudo ufw allow 9001/tcp
sudo ufw allow 9030/tcp

# Or with iptables directly
sudo iptables -A INPUT -p tcp --dport 9001 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9030 -j ACCEPT
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

## Step 4: Start Tor and Enable at Boot

```bash
sudo systemctl enable tor
sudo systemctl start tor
sudo systemctl status tor
```

Check the log to confirm the relay is running:

```bash
sudo tail -f /var/log/tor/notices.log
```

Within a few minutes you should see:

```
[notice] Self-testing indicates your ORPort is reachable from the outside. Excellent.
[notice] Tor has successfully opened a circuit. Looks like client functionality is working.
[notice] Your Tor server's identity key fingerprint is 'MyPrivacyRelay ABCD1234...'
```

The identity fingerprint is your relay's unique identifier in the Tor directory.

## Step 5: Monitor Your Relay

### Check relay status on Metrics Portal

After 3-4 hours, your relay should appear in the Tor relay directory. Check its status:

```
https://metrics.torproject.org/rs.html#search/MyPrivacyRelay
```

Or search by fingerprint from the log file.

### Local monitoring with nyx

Nyx is a terminal status monitor for Tor:

```bash
sudo apt install nyx
sudo nyx
```

Nyx shows real-time bandwidth, circuit count, log messages, and relay configuration in a ncurses interface.

### Check bandwidth usage

```bash
# Bytes transferred by Tor process
sudo cat /proc/$(pidof tor)/net/dev
# Or use vnstat for per-interface monthly totals
sudo apt install vnstat
vnstat -i eth0
```

## Step 6: Relay Flags and Growth Period

New relays go through a **ramp-up period** of approximately 2-3 months before receiving full traffic allocation from the Tor directory authorities. This is intentional — the network builds trust in new relays incrementally.

Relay flags assigned by directory authorities:

- **Running** — relay is reachable
- **Valid** — relay fingerprint is recognized
- **Guard** — relay eligible to be a client's first hop (assigned after ~8 days of stable operation)
- **Fast** — relay has above-median bandwidth
- **Stable** — relay has been online for extended periods
- **HSDir** — can serve hidden service descriptor lookups

You don't need to do anything to earn these flags — they're assigned automatically based on observed behavior.

## Security Considerations

### Separate user account

Tor on Debian/Ubuntu already runs as the `debian-tor` user. Do not run it as root. Verify:

```bash
ps aux | grep tor
# Should show: debian-tor  ...
```

### Keep Tor updated

```bash
sudo apt update && sudo apt upgrade tor
```

Or automate security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### Family key (if running multiple relays)

If you run more than one relay, declare them as a family. This prevents Tor from routing a single circuit through two of your relays:

```
# In torrc on each relay:
MyFamily fingerprint1,fingerprint2,fingerprint3
```

Get the fingerprint from: `sudo cat /var/lib/tor/fingerprint`

## Middle Relay vs Bridge vs Exit: Quick Reference

| Type | Function | Risk level | Who should run it |
|------|----------|------------|-------------------|
| Middle relay | Carries encrypted traffic between nodes | Low | Most people |
| Bridge | Unlisted entry point for censored users | Low-medium | People in uncensored countries |
| Guard/Entry | First hop from Tor clients | Medium | Experienced operators |
| Exit | Final hop to public internet | Higher | Operators with legal support |

Running a middle relay is the most accessible way to contribute to Tor. You handle only encrypted, unreadable traffic and have no visibility into what passes through your node.

## Related Articles

- [Tor Hidden Service Setup Guide Developers](/tor-hidden-service-setup-guide-developers/)
- [How To Use Tor With Specific Application Routing Only](/how-to-use-tor-with-specific-application-routing-only-select/)
- [Tor Browser for Whistleblowers Safety Guide](/tor-browser-for-whistleblowers-safety-guide/)
- [Onionshare Secure File Sharing Over Tor Network Setup](/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Tor Browser Connection Troubleshooting Guide](/tor-browser-connection-troubleshooting-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to set up a tor relay?**

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
