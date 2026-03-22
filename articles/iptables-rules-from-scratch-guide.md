---
layout: default
title: "How to Configure iptables Rules from Scratch"
description: "Build a complete iptables firewall from zero: understand chains, write stateful rules, block unwanted traffic, and persist your config across reboots"
date: 2026-03-22
author: theluckystrike
permalink: /iptables-rules-from-scratch-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Configure iptables Rules from Scratch

iptables is the classic Linux packet filtering framework. Despite nftables becoming the modern successor, iptables remains widespread and is still the default on many LTS distributions. This guide builds a complete, functional firewall from nothing — no pre-existing rules assumed.

## Understanding the Architecture

iptables operates on **tables** containing **chains** containing **rules**.

- **Tables**: filter (packet filtering), nat (address translation), mangle (packet modification)
- **Chains**: INPUT (inbound to host), OUTPUT (outbound from host), FORWARD (packets routed through)
- **Targets**: ACCEPT, DROP, REJECT, LOG, plus custom chain names

Rules are evaluated top-down per chain. The first matching rule wins. If no rule matches, the chain's **default policy** applies.

## Step 1 — Flush All Existing Rules

Start clean. This wipes everything currently loaded:

```bash
# Flush all rules from all chains in the filter table
sudo iptables -F

# Delete all user-defined chains
sudo iptables -X

# Reset all packet and byte counters
sudo iptables -Z

# Same for nat table
sudo iptables -t nat -F
sudo iptables -t nat -X

# Verify — should show empty chains
sudo iptables -L -v --line-numbers
```

**Warning**: if you are on a remote server, flushing rules and setting DROP policies may lock you out. Keep a console session open or set a cron job to flush and accept all before you start.

## Step 2 — Set Default Policies

Decide what happens to packets that match no explicit rule:

```bash
# Deny everything by default — explicitly allow what you need
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

`OUTPUT ACCEPT` is common for workstations. For servers where you want to control outbound too, set it to DROP and add explicit OUTPUT rules.

## Step 3 — Allow Loopback

Local processes communicate via the loopback interface. Always allow it:

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```

## Step 4 — Allow Established and Related Connections

This is the keystone of stateful filtering. Once a connection is established (you initiated it), allow the reply traffic:

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Without this rule, your outbound connections would get no responses.

## Step 5 — Allow SSH

If you're on a remote server, allow SSH before setting the INPUT policy to DROP:

```bash
# Allow SSH from anywhere (restrict to your IP range in production)
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Restrict to a specific IP or CIDR
sudo iptables -A INPUT -p tcp -s 203.0.113.0/24 --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

## Step 6 — Allow Common Services

Add rules for whichever services your machine needs to accept:

```bash
# HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT

# DNS (if this machine is a resolver)
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT

# ICMP ping — useful for diagnostics
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
```

## Step 7 — Rate Limit New Connections

Prevent brute-force attempts on SSH and other services:

```bash
# Allow max 3 new SSH connections per minute per source IP
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --set --name SSH_RATELIMIT

sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH_RATELIMIT \
  -j DROP
```

## Step 8 — Log and Drop Invalid Packets

Log suspicious traffic before dropping it:

```bash
# Log packets with invalid state (common in port scans)
sudo iptables -A INPUT -m conntrack --ctstate INVALID \
  -j LOG --log-prefix "IPTABLES_INVALID: " --log-level 4

sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Log everything dropped (catches all other unwanted traffic)
sudo iptables -A INPUT -j LOG --log-prefix "IPTABLES_DROP: " --log-level 4
```

Logs appear in `/var/log/kern.log` or via `journalctl -k | grep IPTABLES`.

## Step 9 — Complete Ruleset in Script Form

Putting it all together as a reusable script:

```bash
#!/bin/bash
# /etc/iptables/setup-firewall.sh

set -e

IPT="iptables"

# Flush
$IPT -F
$IPT -X
$IPT -Z
$IPT -t nat -F
$IPT -t nat -X

# Default policies
$IPT -P INPUT DROP
$IPT -P FORWARD DROP
$IPT -P OUTPUT ACCEPT

# Loopback
$IPT -A INPUT -i lo -j ACCEPT
$IPT -A OUTPUT -o lo -j ACCEPT

# Established/related
$IPT -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH
$IPT -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# HTTP/HTTPS
$IPT -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT

# ICMP
$IPT -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# SSH rate limiting
$IPT -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --set --name SSH_RATELIMIT
$IPT -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH_RATELIMIT -j DROP

# Drop invalid
$IPT -A INPUT -m conntrack --ctstate INVALID -j DROP

# Log remaining drops
$IPT -A INPUT -j LOG --log-prefix "IPTABLES_DROP: " --log-level 4

echo "Firewall configured."
```

Make it executable and run it:

```bash
sudo chmod +x /etc/iptables/setup-firewall.sh
sudo /etc/iptables/setup-firewall.sh
```

## Step 10 — Persist Rules Across Reboots

Rules are in-memory only by default. Save and restore them:

```bash
# Save current rules
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# Rules are stored at:
# /etc/iptables/rules.v4
# /etc/iptables/rules.v6

# Restore manually anytime
sudo netfilter-persistent reload
```

Alternatively, load your setup script from a systemd unit:

```ini
# /etc/systemd/system/firewall.service
[Unit]
Description=Custom iptables firewall
Before=network.target

[Service]
Type=oneshot
ExecStart=/etc/iptables/setup-firewall.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now firewall.service
```

## Verifying Your Rules

```bash
# Show rules with packet counts
sudo iptables -L -v --line-numbers

# Show rules as iptables commands (useful for auditing)
sudo iptables-save

# Test a specific port from another host
nc -zv YOUR_SERVER_IP 22
nc -zv YOUR_SERVER_IP 80
```

## Related Reading

- [VPN Kill Switch with iptables on Linux](/privacy-tools-guide/vpn-kill-switch-linux-iptables-setup/)
- [How VPN Interacts with Firewall Rules](/privacy-tools-guide/how-vpn-interacts-with-firewall-rules-iptables-nftables-guide/)
- [How to Set Up Cilium for Network Security](/privacy-tools-guide/cilium-network-security-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
