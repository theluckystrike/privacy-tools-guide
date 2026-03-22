---
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

### Step 1: Generate an SSH Key Pair

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

Why Ed25519 over RSA? Ed25519 uses elliptic curve cryptography on Curve25519. The keys are 256 bits, produce 64-byte signatures, and are significantly faster to verify than RSA-2048. Ed25519 is also resistant to side-channel attacks that affect RSA implementations. If you must use RSA for compatibility with older systems, use RSA-4096 at minimum.

### Step 2: Back Up the Default Config

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

If anything goes wrong, restore it:

```bash
sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### Step 3: The Hardened sshd_config

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

## Why Each Cryptographic Setting Matters

The cipher list deserves explanation. Legacy algorithms still present in many default OpenSSH installs include `arcfour` (RC4, broken), `3des-cbc` (triple DES with CBC mode — vulnerable to SWEET32), and `aes128-cbc` (CBC mode susceptible to BEAST and Lucky13 attacks).

The `etm` suffix on MAC algorithms stands for Encrypt-Then-MAC, which is the correct ordering. Older `hmac-sha2-512` without `etm` uses MAC-Then-Encrypt, which enables padding oracle attacks. Always prefer ETM variants.

The `KexAlgorithms` list removes `diffie-hellman-group14-sha256` and all `diffie-hellman-group1` or `group14` with SHA-1. Group 14 uses 2048-bit DH which is borderline acceptable, but groups 16 (4096-bit) and 18 (8192-bit) with SHA-512 are definitively modern. The `curve25519` options are preferred when both sides support them — they are faster and resist attacks that target traditional finite-field DH.

### Step 4: Verify from a New Terminal

Before closing your existing session, open a new one and test:

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 user@your-server-ip
```

If it connects, the hardened config is working. If not, restore the backup from your existing session.

### Step 5: Update Your Firewall

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

### Step 6: Add a Legal Warning Banner

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

### Step 7: Install and Configure fail2ban

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

### fail2ban Tuning for High-Traffic Servers

On servers that receive many legitimate connections from dynamic IPs (CI/CD systems, monitoring agents), aggressive ban settings cause operational pain. Use `ignoreip` to whitelist known good addresses:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 203.0.113.0/24
bantime  = 24h
findtime = 5m
maxretry = 3
```

You can also use `bantime.increment = true` with `bantime.factor = 2` to double ban duration on repeat offenders — a useful escalating deterrent for persistent scanners.

### Step 8: Restrict SSH Access by IP (Optional but Effective)

If you connect from a known static IP, restrict access at the firewall level:

```bash
# UFW — only allow SSH from your IP
sudo ufw allow from 203.0.113.10 to any port 2222 proto tcp
sudo ufw deny 2222/tcp
sudo ufw reload
```

For dynamic IPs, use a VPN or jump host instead.

### Step 9: Use SSH Certificates Instead of Authorized Keys

For environments with multiple servers and multiple users, per-user `authorized_keys` files become difficult to manage. SSH certificates offer a more scalable alternative.

Generate a Certificate Authority (CA) key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ca_key -C "Internal SSH CA"
```

Sign a user's public key:

```bash
ssh-keygen -s ~/.ssh/ca_key -I "mike@example.com" -n deploy,admin -V +30d ~/.ssh/id_ed25519.pub
```

The `-V +30d` flag makes the certificate expire in 30 days, enforcing regular rotation. The `-n` flag specifies which principals (usernames) the certificate is valid for.

On each server, trust the CA rather than individual keys:

```bash
# Add to sshd_config
TrustedUserCAKeys /etc/ssh/ca.pub

# Disable per-user authorized_keys if going certificate-only
AuthorizedKeysFile none
```

Copy the CA public key to all servers:

```bash
sudo cp ~/.ssh/ca_key.pub /etc/ssh/ca.pub
```

This approach means revoking access requires only removing the user from valid principals at certificate signing time — no need to touch individual server `authorized_keys` files.

### Step 10: Audit Current Connections

View active SSH sessions:

```bash
who
ss -tnp | grep :2222
```

Check recent authentication attempts:

```bash
sudo journalctl -u sshd --since "24 hours ago" | grep -E "(Failed|Accepted|Invalid)"
```

### Step 11: SSH Key Management

List authorized keys on the server:

```bash
cat ~/.ssh/authorized_keys
```

Remove a key by deleting its line. To audit key fingerprints:

```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```

For servers with many users, consider `AuthorizedKeysCommand` to fetch keys from a central directory service rather than per-user files.

### Step 12: Periodic Config Audit

Run `ssh-audit` (a Python tool) monthly to check your server configuration against current recommendations:

```bash
pip install ssh-audit
ssh-audit localhost -p 2222
```

The tool outputs a color-coded report of algorithms, highlighting deprecated entries and suggesting replacements. It tests both server-side configuration and the actual negotiated algorithms — catching cases where sshd_config looks correct but system-level defaults override individual settings.

### Step 13: Hardening Checklist

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
- ssh-audit run and clean
- Certificate-based auth considered for multi-server environments

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to complete this setup?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
- [How to Harden SSH Server Configuration](/privacy-tools-guide/how-to-harden-ssh-server-configuration/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)
- [How To Prepare Ssh Key And Server Access Documentation](/privacy-tools-guide/how-to-prepare-ssh-key-and-server-access-documentation-for-t/)
- [How To Use Ssh Tunneling For Encrypted Communication](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
