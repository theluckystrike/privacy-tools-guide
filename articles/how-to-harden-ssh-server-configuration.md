---
layout: default
title: "How to Harden SSH Server Configuration"
description: "Secure SSH with key-only authentication, Ed25519 keys, port knocking, fail2ban, and system-level lockdown to eliminate brute-force exposure"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-harden-ssh-server-configuration/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A default OpenSSH installation accepts password authentication, root logins, and listens on port 22 — the first three things attackers target. This guide hardens SSH systematically: key authentication, cipher hardening, access controls, and automated blocking of brute-force attempts.

## Step 1: Generate Ed25519 Keys (Client Side)

```bash
# Generate Ed25519 key pair (smaller and faster than RSA-4096)
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519

# Use a strong passphrase — it protects the private key at rest

# If you need RSA (legacy compatibility):
ssh-keygen -t rsa -b 4096 -C "your@email.com" -f ~/.ssh/id_rsa

# View your public key for upload to servers
cat ~/.ssh/id_ed25519.pub
```

## Step 2: Deploy the Public Key to the Server

```bash
# Copy public key to server (preferred method)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   cat >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"

# Verify you can log in with key before disabling passwords
ssh -i ~/.ssh/id_ed25519 user@server
```

## Step 3: Harden sshd_config

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply these settings:

```conf
# Port (change from default 22 to reduce automated scan noise)
# Choose a port above 1024 that doesn't conflict with other services
Port 2222

# Protocol version (OpenSSH 7+ only supports v2 by default, but be explicit)
Protocol 2

# Key algorithms — prefer Ed25519 and ECDSA, disable old ones
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_ecdsa_key
# Remove or comment out RSA and DSA host keys if not needed

# Authentication
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# Public key authentication
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Key exchange algorithms — modern, strong algorithms only
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Access control
AllowUsers youruser adminuser
# Or restrict to specific IPs:
# Match User deploy
#   AllowedUsers 10.0.0.0/24

# Session settings
ClientAliveInterval 300
ClientAliveCountMax 2
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 20

# Disable features you don't need
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
PrintLastLog yes

# Logging
LogLevel VERBOSE
SyslogFacility AUTH
```

Validate and reload:
```bash
# Syntax check BEFORE reloading (critical — don't lock yourself out)
sudo sshd -t

# Reload
sudo systemctl reload sshd

# Keep your current session open and test in a NEW terminal window
ssh -p 2222 youruser@server
```

## Step 4: Regenerate Host Keys

The default host keys generated at installation may use weak parameters. Regenerate:

```bash
# Remove old host keys
sudo rm /etc/ssh/ssh_host_*

# Generate new strong keys
sudo ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ""
sudo ssh-keygen -t ecdsa -b 521 -f /etc/ssh/ssh_host_ecdsa_key -N ""
# RSA 4096 for legacy client compatibility
sudo ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_host_rsa_key -N ""

sudo systemctl restart sshd
```

Clients will see a "host key changed" warning — clear known_hosts and reconnect:
```bash
ssh-keygen -R server_ip
ssh-keygen -R server_hostname
```

## Step 5: Install and Configure fail2ban

fail2ban monitors auth.log and blocks IPs that exceed a threshold of failed logins:

```bash
sudo apt install fail2ban

# Create local override (never edit the main .conf file)
sudo tee /etc/fail2ban/jail.local > /dev/null <<'EOF'
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 3
backend = systemd

[sshd]
enabled = true
port = 2222
logpath = %(sshd_log)s
maxretry = 3
bantime = 24h
EOF

sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Check current bans:
```bash
sudo fail2ban-client status sshd
# Shows currently banned IPs

# Manually unban an IP (if you ban yourself)
sudo fail2ban-client set sshd unbanip YOUR_IP
```

## Step 6: UFW Rules for SSH

```bash
# Allow SSH only from a specific IP (strongest option)
sudo ufw allow from YOUR_OFFICE_IP to any port 2222 proto tcp

# Or rate-limit if you need access from any IP
sudo ufw limit 2222/tcp comment "SSH rate limited"

sudo ufw enable
sudo ufw status
```

## Step 7: Two-Factor Authentication (Optional)

Add TOTP 2FA on top of key authentication for additional protection:

```bash
sudo apt install libpam-google-authenticator

# Run as the user who will log in
google-authenticator
# Answer yes to all prompts
# Save the emergency scratch codes

# Edit /etc/pam.d/sshd
# Add BEFORE the @include common-auth line:
# auth required pam_google_authenticator.so

# Edit /etc/ssh/sshd_config
# AuthenticationMethods publickey,keyboard-interactive
# ChallengeResponseAuthentication yes

sudo systemctl restart sshd
```

Now SSH requires both your private key AND the TOTP code.

## Verify Your Hardened Configuration

```bash
# Check what ciphers the server advertises
ssh -vvv user@server 2>&1 | grep -i "kex\|cipher\|hmac"

# Audit with ssh-audit
pip3 install ssh-audit
ssh-audit server:2222

# Check for common vulnerabilities
# ssh-audit grades your configuration and lists specific issues
```

## SSH Client Configuration

Harden your local SSH client too. Edit `~/.ssh/config`:

```conf
Host *
    # Use Ed25519 keys preferentially
    IdentityFile ~/.ssh/id_ed25519

    # Strong algorithms only
    KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
    Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

    # Security
    VerifyHostKeyDNS yes
    AddKeysToAgent yes
    StrictHostKeyChecking ask

    # Connection reuse (faster connections after first)
    ControlMaster auto
    ControlPath ~/.ssh/controlmasters/%r@%h:%p
    ControlPersist 10m
```

## Related Articles

- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
- [How To Use Ssh Tunneling For Encrypted Communication](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [SSH Server Hardening Config Guide](/privacy-tools-guide/ssh-server-hardening-config-guide)
- [How to Use YubiKey for SSH Authentication](/privacy-tools-guide/articles/how-to-use-yubikey-for-ssh-authentication-guide/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
