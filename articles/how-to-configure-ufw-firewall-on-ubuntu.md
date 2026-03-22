---
layout: default
title: "How to Configure UFW Firewall on Ubuntu"
description: "Set up UFW on Ubuntu with practical rules for SSH, web servers, rate limiting, and logging to harden your server perimeter"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-configure-ufw-firewall-on-ubuntu/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

UFW (Uncomplicated Firewall) is a front-end for iptables that ships with Ubuntu. It gives you a readable interface for managing packet filtering without writing raw iptables rules. This guide covers practical hardening: default deny, SSH protection, web server rules, and logging.

## Key Takeaways

- **SRC=203.0.113.99 DST=10.0.0.1 \ LEN=44**: TTL=244 ID=54321 DF PROTO=TCP SPT=44231 DPT=3306 FLAGS=S ``` ## Step 9: IPv6 Support Ensure IPv6 rules are also applied.
- **Topics covered**: prerequisites, check ufw status, step 1: set default policies
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Prerequisites

- Ubuntu 20.04 or later
- Root or sudo access
- An active SSH session (read the SSH section before enabling UFW to avoid locking yourself out)

## Check UFW Status

```bash
sudo ufw status verbose
# If inactive, no rules are applied yet
```

## Step 1: Set Default Policies

The most important step: deny all incoming traffic and allow all outgoing by default.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny forward
```

Do not enable UFW yet — add SSH rules first.

## Step 2: Allow SSH Before Enabling

If you lock yourself out of SSH on a remote server, you will need console access to recover.

```bash
# Allow SSH on default port 22
sudo ufw allow ssh

# Or specify explicitly
sudo ufw allow 22/tcp

# If you run SSH on a custom port (e.g., 2222)
sudo ufw allow 2222/tcp
```

For extra protection, limit SSH to a specific source IP:
```bash
# Only allow SSH from your office IP
sudo ufw allow from 203.0.113.50 to any port 22 proto tcp
```

## Step 3: Enable UFW

```bash
sudo ufw enable
# Command may disrupt existing connections. Proceed with operation (y|n)? y

sudo ufw status verbose
```

Output should show:
```
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)
```

## Step 4: Common Service Rules

### Web Server

```bash
# HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Or use the application profile
sudo ufw allow 'Nginx Full'
sudo ufw allow 'Apache Full'
```

### DNS (if running a resolver)

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

### Mail Server

```bash
sudo ufw allow 25/tcp    # SMTP
sudo ufw allow 587/tcp   # Submission
sudo ufw allow 993/tcp   # IMAPS
sudo ufw allow 465/tcp   # SMTPS
```

### PostgreSQL (only from app server)

```bash
sudo ufw allow from 10.0.0.5 to any port 5432 proto tcp
```

## Step 5: Rate Limit SSH

UFW has a built-in rate limit that blocks an IP after 6 connection attempts within 30 seconds.

```bash
# Replace the plain allow rule with a rate-limited one
sudo ufw delete allow 22/tcp
sudo ufw limit ssh

# Verify
sudo ufw status numbered
```

This mitigates brute-force attacks without needing fail2ban for basic SSH protection.

## Step 6: Block Known Bad Ranges

Block entire CIDR ranges when you want to deny traffic from a region or known malicious ASN:

```bash
# Block a range
sudo ufw deny from 192.0.2.0/24

# Block and log
sudo ufw deny log from 198.51.100.0/24
```

Note: UFW processes rules in order. A `deny` before a more specific `allow` will block that traffic. List rules with numbers to check ordering:

```bash
sudo ufw status numbered
```

Delete a rule by number:
```bash
sudo ufw delete 3
```

## Step 7: Application Profiles

UFW ships with application profiles in `/etc/ufw/applications.d/`. List available profiles:

```bash
sudo ufw app list
sudo ufw app info 'Nginx Full'
```

Create a custom profile for your application:

```bash
sudo tee /etc/ufw/applications.d/myapp <<EOF
[MyApp]
title=My Application
description=Custom app on port 8080
ports=8080/tcp
EOF

sudo ufw app update MyApp
sudo ufw allow MyApp
```

## Step 8: Enable Logging

```bash
# Low logging: blocked packets only
sudo ufw logging on

# Medium: includes new connections
sudo ufw logging medium

# High: all packets (very verbose)
sudo ufw logging high
```

Logs appear in `/var/log/ufw.log` and also via syslog/journalctl:

```bash
sudo tail -f /var/log/ufw.log
sudo journalctl -k --grep="UFW" -f
```

Sample blocked packet log entry:
```
[UFW BLOCK] IN=eth0 OUT= MAC=... SRC=203.0.113.99 DST=10.0.0.1 \
  LEN=44 TTL=244 ID=54321 DF PROTO=TCP SPT=44231 DPT=3306 FLAGS=S
```

## Step 9: IPv6 Support

Ensure IPv6 rules are also applied. Open `/etc/default/ufw` and confirm:

```bash
sudo grep IPV6 /etc/default/ufw
# Should show: IPV6=yes
```

If it is `no`, change it and reload:
```bash
sudo ufw disable && sudo ufw enable
```

UFW will then apply matching rules to both IPv4 and IPv6 interfaces automatically.

## Step 10: Allow Docker Containers

Docker bypasses UFW by default, writing directly to iptables. To fix this without patching Docker:

Edit `/etc/ufw/after.rules` and add before `COMMIT`:

```bash
# Allow forwarding for Docker
-A ufw-before-forward -i docker0 -j ACCEPT
-A ufw-before-forward -o docker0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

A cleaner solution is to bind Docker to `127.0.0.1` in `/etc/docker/daemon.json`:
```json
{
  "iptables": false
}
```

Then manage all exposure through UFW explicitly.

## Useful Management Commands

```bash
# Show rules with line numbers
sudo ufw status numbered

# Reset all rules (dangerous — disables UFW)
sudo ufw reset

# Reload after manual edits
sudo ufw reload

# Delete a rule by specification
sudo ufw delete allow 8080/tcp

# Check before enabling (dry run)
sudo ufw --dry-run enable
```

## Related Reading

- [How to Detect Keyloggers on Your System](/privacy-tools-guide/how-to-detect-keyloggers-on-your-system/)
- [Securing Docker Containers Best Practices](/privacy-tools-guide/securing-docker-containers-best-practices/)
- [How to Set Up Nebula Mesh VPN for Teams](/privacy-tools-guide/how-to-set-up-nebula-mesh-vpn-for-teams/)
- [How to Set Up AppArmor Profiles on Ubuntu](/privacy-tools-guide/apparmor-profiles-ubuntu-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

## Related Articles

- [How to Harden SSH Server Configuration](/privacy-tools-guide/how-to-harden-ssh-server-configuration/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)
- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
- [How To Use Ssh Tunneling For Encrypted Communication](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [SSH Server Hardening Config Guide](/privacy-tools-guide/ssh-server-hardening-config-guide)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
