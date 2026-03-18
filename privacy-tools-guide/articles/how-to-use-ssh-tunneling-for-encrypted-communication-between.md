---
layout: default
title: "How to Use SSH Tunneling for Encrypted Communication Between Devices"
description: "A practical guide to securing device-to-device communication using SSH tunnels. Learn local, remote, and dynamic port forwarding with real-world examples."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-ssh-tunneling-for-encrypted-communication-between/
---

SSH tunneling creates encrypted channels between devices, protecting your data from interception. Whether you're accessing a service on a remote machine or forwarding traffic through a jump host, SSH tunnels provide a secure alternative to unencrypted protocols.

This guide covers three types of SSH port forwarding with practical examples you can implement immediately.

## Understanding SSH Tunnel Types

SSH offers three port forwarding mechanisms: local, remote, and dynamic. Each serves different use cases.

### Local Port Forwarding

Local port forwarding redirects traffic from your local machine through an SSH connection to a destination server. The syntax:

```bash
ssh -L [local_port]:[destination_host]:[destination_port] [user]@[ssh_server]
```

Example: Access a MySQL database on a remote server securely:

```bash
ssh -L 3307:localhost:3306 user@remote-server.com
```

This maps local port 3307 to the remote MySQL service. Connect your application to `localhost:3307`, and the traffic flows encrypted through the SSH tunnel to the remote server's port 3306.

For a more practical scenario, imagine you need to access a web interface on a remote server that only accepts local connections:

```bash
ssh -L 8080:localhost:80 user@remote-server.com
```

Now open `http://localhost:8080` in your browser to reach the remote web service through an encrypted tunnel.

### Remote Port Forwarding

Remote port forwarding does the opposite—it makes a local service accessible from the remote SSH server. This is useful when the remote server cannot directly reach your local machine due to NAT or firewalls.

```bash
ssh -R [remote_port]:[local_host]:[local_port] [user]@[ssh_server]
```

Example: Expose a local development server to the internet via a remote SSH host:

```bash
ssh -R 8080:localhost:3000 user@vps-server.com
```

After establishing this tunnel, anyone connecting to `vps-server.com:8080` reaches your local service on port 3000. The connection travels encrypted from the client through the SSH server to your local machine.

This technique works well for testing webhooks or sharing local development environments with collaborators.

### Dynamic Port Forwarding

Dynamic port forwarding turns your SSH client into a SOCKS proxy, routing all traffic through the SSH server. This encrypts your web browsing and other applications without configuring individual services:

```bash
ssh -D [local_port] -N [user]@[ssh_server]
```

Example: Create a local SOCKS5 proxy on port 1080:

```bash
ssh -D 1080 -N user@proxy-server.com
```

Configure your browser or system settings to use `localhost:1080` as a SOCKS5 proxy. All traffic now routes through the SSH server, encrypting your connection and masking your IP address.

For command-line tools that support SOCKS proxies, set the environment variable:

```bash
export ALL_PROXY="socks5://localhost:1080"
curl https://example.com
```

## Persisting SSH Tunnels

SSH tunnels close when the connection drops. For production or long-running use cases, you need persistent tunnels.

### Using autossh

The `autossh` tool automatically restarts tunnels when they fail:

```bash
autossh -M 20000 -f -N -D 1080 user@proxy-server.com
```

- `-M 20000`: Monitoring port for health checks
- `-f`: Run in background
- `-N`: Don't execute remote commands (port forwarding only)

Install on macOS with Homebrew:

```bash
brew install autossh
```

On Linux:

```bash
sudo apt install autossh
```

### Systemd Service for Persistent Tunnels

Create a systemd service to start tunnels automatically on boot. Create `/etc/systemd/system/ssh-tunnel.service`:

```ini
[Unit]
Description=SSH Tunnel Service
After=network.target

[Service]
Type=simple
User=yourusername
Restart=always
RestartSec=10
ExecStart=/usr/bin/autossh -M 20000 -N -D 1080 user@proxy-server.com

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ssh-tunnel
sudo systemctl start ssh-tunnel
```

Check status:

```bash
sudo systemctl status ssh-tunnel
```

## Security Considerations

SSH tunnels protect data in transit, but additional precautions strengthen your setup.

### Key-Based Authentication

Disable password authentication and use SSH keys:

```bash
# Generate ED25519 key (recommended)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

Edit `/etc/ssh/sshd_config` on the server:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

### Restrict Tunnel Permissions

Limit which ports users can forward by editing `/etc/ssh/sshd_config`:

```
AllowTcpForwarding yes
PermitOpen localhost:8080
```

The `PermitOpen` directive restricts remote forwarding to specific destinations.

### Connection Timeout

Prevent idle connections from hanging indefinitely. Add to your SSH config `~/.ssh/config`:

```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

This sends keepalive packets every 60 seconds, detecting dead connections quickly.

## Practical Example: Secure Database Access

Consider a common scenario: accessing a production PostgreSQL database from your local machine through a bastion host.

Network topology:
- Your laptop: local machine
- Bastion server: `bastion.example.com` (publicly accessible)
- Database server: `db.internal.example.com` (only accessible from bastion)

Create the tunnel:

```bash
ssh -L 5433:db.internal.example.com:5432 user@bastion.example.com
```

Connect your database client:

```bash
psql -h localhost -p 5433 -U dbuser -d production
```

The connection path: your laptop → bastion (encrypted) → database server. The database traffic never travels unencrypted over the internet.

## Conclusion

SSH tunneling provides encrypted communication between devices without requiring VPN infrastructure. Local port forwarding handles service access, remote forwarding exposes local services externally, and dynamic forwarding creates personal SOCKS proxies. Combine these with autossh and systemd for persistent, automated tunnels.

For developers working across multiple environments or power users concerned about network surveillance, SSH tunnels offer a lightweight security layer that works over existing SSH access—no additional software required.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
