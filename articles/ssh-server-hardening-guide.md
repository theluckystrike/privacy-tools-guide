---
layout: default
title: "SSH Server Hardening Guide"
description: "Step-by-step SSH server hardening for Linux - disable password auth, configure key-only login, change default port, set up fail2ban, and audit active ses..."
date: 2026-03-21
author: theluckystrike
permalink: /ssh-server-hardening-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Every publicly accessible Linux server with SSH open on port 22 is scanned by automated bots within minutes of going online. These bots attempt thousands of common username/password combinations, looking for weak credentials. Hardening SSH closes the most common attack vectors and dramatically reduces your server's exposure.

This guide covers the essential hardening steps for OpenSSH on a Linux server, ordered from highest to lowest impact.

Step 0 - Back Up Working Access Before Starting

Before making any changes, ensure you have an alternative way to access the server (console access through your hosting provider, a second SSH session, etc.). Locking yourself out is a real risk when changing SSH configuration.

Step 1 - Set Up SSH Key Authentication

Password authentication should be completely disabled. SSH key authentication is both more secure and more convenient.

Generate a key pair on your local machine

```bash
Generate an Ed25519 key (preferred) with a comment
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519

Or RSA 4096 if Ed25519 isn't supported by your target
ssh-keygen -t rsa -b 4096 -C "your@email.com" -f ~/.ssh/id_rsa
```

Enter a strong passphrase. This protects the private key if your local machine is compromised.

Copy the public key to the server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server-ip

Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server-ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Test key authentication before disabling passwords:
```bash
ssh -i ~/.ssh/id_ed25519 user@server-ip
```

Step 2 - Harden sshd_config

Edit the main SSH server configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply these settings:

```
Disable password authentication entirely
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

Disable root login
PermitRootLogin no

Only allow specific users or groups to SSH (replace with your username)
AllowUsers yourusername

Disable empty passwords
PermitEmptyPasswords no

Use only modern key exchange algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

Restrict ciphers to strong options
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

Restrict MACs
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com

Disable X11 forwarding (not needed on servers)
X11Forwarding no

Disable TCP forwarding if not needed
AllowTcpForwarding no

Disable agent forwarding
AllowAgentForwarding no

Limit authentication attempts
MaxAuthTries 3

Disconnect idle sessions after 10 minutes
ClientAliveInterval 600
ClientAliveCountMax 0

Log level (INFO is appropriate for most; VERBOSE adds connection fingerprints)
LogLevel VERBOSE

Disable host-based authentication
HostbasedAuthentication no
IgnoreRhosts yes

Disable .rhosts files
RhostsRSAAuthentication no

Only use SSH protocol 2
Protocol 2
```

Apply the changes:

```bash
Test configuration before reloading
sudo sshd -t

If no errors:
sudo systemctl reload sshd
```

Step 3 - Change the Default SSH Port

Moving SSH off port 22 stops automated scans from the vast majority of bots (which only scan well-known ports). This is not a security control. a targeted attacker will port-scan and find your service. but it dramatically reduces noise in your logs and lowers the attack surface for opportunistic attacks.

In `sshd_config`:
```
Port 2222
```

Or use a port in the 49152-65535 range to avoid conflicts with other services.

Update your firewall:
```bash
UFW
sudo ufw allow 2222/tcp
sudo ufw delete allow 22/tcp

iptables
sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT
```

Update your SSH client config to use the new port:
```
~/.ssh/config
Host myserver
    HostName server-ip
    Port 2222
    User yourusername
    IdentityFile ~/.ssh/id_ed25519
```

Step 4 - Install and Configure fail2ban

fail2ban monitors SSH logs and bans IPs that have too many failed authentication attempts:

```bash
sudo apt install fail2ban

Create a local jail configuration
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
banaction = iptables-multiport

[sshd]
enabled = true
port    = 2222    # Your SSH port
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime  = 24h
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

Check ban status
sudo fail2ban-client status sshd
```

Step 5 - Configure a Host-Based Firewall

Only allow SSH connections from trusted IP addresses where possible:

```bash
UFW - allow SSH only from your home IP
sudo ufw allow from 203.0.113.10 to any port 2222 proto tcp

Deny SSH from all other IPs
sudo ufw deny 2222/tcp
```

If you have a dynamic IP, this isn't practical. In that case, rely on fail2ban and key-only auth.

Step 6 - Audit Current SSH Sessions and Authorized Keys

Periodically check who is logged in and what keys have access:

```bash
Check currently logged-in users
who
w

Check active SSH sessions
ss -tnp | grep ssh
or
sudo netstat -tnp | grep sshd

Review all authorized keys
find /home -name "authorized_keys" -exec cat {} \;
sudo cat /root/.ssh/authorized_keys

Check SSH login history
last | grep sshd | head -20

Check failed login attempts
sudo grep "Failed password\|Invalid user" /var/log/auth.log | tail -30

Check banned IPs from fail2ban
sudo fail2ban-client status sshd | grep "Banned IP"
```

Step 7 - Enable SSH Certificates (Advanced)

For environments with multiple servers and users, SSH certificates are more manageable than distributing public keys:

```bash
Create a Certificate Authority for your infrastructure
ssh-keygen -t ed25519 -f ssh_ca -C "Infrastructure CA"

Sign a user's public key to create a certificate
ssh-keygen -s ssh_ca -I "username" -n "username" -V +52w ~/.ssh/id_ed25519.pub

Configure sshd to trust the CA
echo "TrustedUserCAKeys /etc/ssh/ssh_ca.pub" | sudo tee -a /etc/ssh/sshd_config
sudo cp ssh_ca.pub /etc/ssh/ssh_ca.pub
sudo systemctl reload sshd
```

Now any key signed by your CA can authenticate without being individually added to `authorized_keys` on each server. Revocation is handled via a `RevokedKeys` file.

Verify the Hardening

```bash
Test that password auth is disabled (should fail)
ssh -o PubkeyAuthentication=no user@server-ip -p 2222

Run an SSH security audit
Install ssh-audit
pip3 install ssh-audit
ssh-audit server-ip

Or use the online tool at ssh-audit.com
```

A well-hardened server should score an "A" or "A+" on ssh-audit with no flagged algorithms.

Related Articles

- [SSH Server Hardening Config Guide](/ssh-server-hardening-config-guide)
- [How to Harden SSH Server Configuration](/how-to-harden-ssh-server-configuration/)
- [How To Prepare Ssh Key And Server Access Documentation](/how-to-prepare-ssh-key-and-server-access-documentation-for-t/)
- [How To Use Ssh Tunneling For Encrypted Communication](/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [How to Set Up a Password Manager for Home Server SSH](/how-to-set-up-password-manager-for-home-server-ssh-keys/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

Frequently Asked Questions

How long does it take to complete this setup?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.
```
```
{% endraw %}
