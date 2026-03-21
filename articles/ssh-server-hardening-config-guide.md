---
layout: default
title: "SSH Server Hardening Config Guide"
description: "Step-by-step guide to hardening sshd_config on Linux servers. Disable root login, enforce key auth, restrict ciphers, and set up fail2ban"
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: ssh-server-hardening-config-guide
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A default SSH installation accepts password logins, allows root access, and supports legacy ciphers that have known weaknesses. Every internet-facing server running stock SSH config is one brute-force attempt away from disaster.

This guide walks through locking down `sshd_config` to a minimal, defensible state — key-only authentication, no root login, restricted algorithms, and basic rate limiting.

## Prerequisites

- A Linux server (Debian/Ubuntu or RHEL/Fedora)
- A non-root user with `sudo` privileges
- An SSH key pair already generated (if not, see below)

**Do not lock yourself out.** Before making changes, open a second terminal session and keep it connected until you verify the new config works.

## Generate an SSH Key Pair

If you don't have a key pair:

```bash
# Ed25519 is preferred — smaller, faster, more secure than RSA 2048
ssh-keygen -t ed25519 -C "yourname@host" -f ~/.ssh/id_ed25519

# Copy the public key to the server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@your-server-ip
```

Verify you can log in with the key before disabling password authentication:

```bash
ssh -i ~/.ssh/id_ed25519 user@your-server-ip
```

## Back Up the Default Config

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

If anything goes wrong, restore it:

```bash
sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
sudo systemctl restart sshd
```

## The Hardened sshd_config

Open the file:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply these settings. Comment out or remove conflicting defaults:

```
# ============================================================
# Port and Protocol
# ============================================================
Port 2222                        # Change from default 22
AddressFamily inet               # IPv4 only; use 'any' if you need IPv6
ListenAddress 0.0.0.0

# ============================================================
# Authentication
# ============================================================
PermitRootLogin no               # Never allow direct root SSH
PasswordAuthentication no        # Keys only
ChallengeResponseAuthentication no
UsePAM no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                   # Limit guessing attempts per connection
LoginGraceTime 30                # Seconds before unauthenticated session drops

# ============================================================
# User / Group Restrictions
# ============================================================
AllowUsers deploy admin          # Whitelist specific users only
# AllowGroups sshusers           # Alternative: allow by group

# ============================================================
# Session Hardening
# ============================================================
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no            # Set to 'yes' only if you use tunnels
PermitTunnel no
PrintLastLog yes
Banner /etc/issue.net            # Legal warning banner

# ============================================================
# Idle Timeout
# ============================================================
ClientAliveInterval 300          # Send keepalive every 5 minutes
ClientAliveCountMax 2            # Drop after 2 missed keepalives (10 min)

# ============================================================
# Cryptography — Restrict to Modern Algorithms
# ============================================================
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# ============================================================
# Logging
# ============================================================
SyslogFacility AUTH
LogLevel VERBOSE                 # Logs key fingerprints; useful for audits
```

Save and test the configuration syntax:

```bash
sudo sshd -t
```

If there are no errors, restart SSH:

```bash
sudo systemctl restart sshd
```

## Verify from a New Terminal

Before closing your existing session, open a new one and test:

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 user@your-server-ip
```

If it connects, the hardened config is working. If not, restore the backup from your existing session.

## Update Your Firewall

If you changed the port, update your firewall rules:

```bash
# UFW
sudo ufw allow 2222/tcp
sudo ufw delete allow 22/tcp
sudo ufw reload

# firewalld
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

## Add a Legal Warning Banner

Create the banner file:

```bash
sudo nano /etc/issue.net
```

Add something like:

```
Authorized access only. All activity is logged and monitored.
Unauthorized access will be prosecuted.
```

The `Banner` directive in `sshd_config` already points to this file.

## Install and Configure fail2ban

fail2ban blocks IPs after repeated failed login attempts:

```bash
# Debian/Ubuntu
sudo apt install fail2ban

# RHEL/Fedora
sudo dnf install fail2ban
```

Create a local jail configuration:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3

[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 24h
```

Enable and start fail2ban:

```bash
sudo systemctl enable --now fail2ban
```

Check active bans:

```bash
sudo fail2ban-client status sshd
```

## Restrict SSH Access by IP (Optional but Effective)

If you connect from a known static IP, restrict access at the firewall level:

```bash
# UFW — only allow SSH from your IP
sudo ufw allow from 203.0.113.10 to any port 2222 proto tcp
sudo ufw deny 2222/tcp
sudo ufw reload
```

For dynamic IPs, use a VPN or jump host instead.

## Audit Current Connections

View active SSH sessions:

```bash
who
ss -tnp | grep :2222
```

Check recent authentication attempts:

```bash
sudo journalctl -u sshd --since "24 hours ago" | grep -E "(Failed|Accepted|Invalid)"
```

## SSH Key Management

List authorized keys on the server:

```bash
cat ~/.ssh/authorized_keys
```

Remove a key by deleting its line. To audit key fingerprints:

```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```

For servers with many users, consider `AuthorizedKeysCommand` to fetch keys from a central directory service rather than per-user files.

## Hardening Checklist

- Root login disabled (`PermitRootLogin no`)
- Password authentication disabled
- Non-default port configured
- User whitelist in place (`AllowUsers`)
- Legacy ciphers removed
- Idle timeout configured
- fail2ban running and protecting the SSH port
- Firewall restricting SSH access
- Warning banner enabled
- Log level set to VERBOSE


## Related Articles

- [How To Prepare Ssh Key And Server Access Documentation For T](/privacy-tools-guide/how-to-prepare-ssh-key-and-server-access-documentation-for-t/)
- [How to Set Up a Password Manager for Home Server SSH Keys](/privacy-tools-guide/how-to-set-up-password-manager-for-home-server-ssh-keys/)
- [Complete Guide To Operating System Hardening For Extreme Pri](/privacy-tools-guide/complete-guide-to-operating-system-hardening-for-extreme-pri/)
- [How To Use Ssh Tunneling For Encrypted Communication Between](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [Linux Desktop Privacy Hardening Guide](/privacy-tools-guide/linux-desktop-privacy-hardening-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
