---
layout: default
title: "How To Use Ssh Tunneling For Encrypted Communication Between"
description: "Learn how to create secure SSH tunnels to encrypt communication between devices. This guide covers local, remote, and dynamic port forwarding with."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-ssh-tunneling-for-encrypted-communication-between/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

SSH tunneling creates encrypted pathways between devices, securing data that would otherwise travel in plaintext. Whether you're accessing a database on a remote server, protecting web traffic on public WiFi, or forwarding services across networks, SSH tunnels provide a lightweight alternative to VPNs. This guide walks through the three main tunnel types with real examples you can apply immediately.

## Understanding SSH Tunnels

An SSH tunnel forwards network traffic through an encrypted SSH connection. The SSH protocol already encrypts your terminal session—tunneling extends that encryption to arbitrary ports and services. This means any service using TCP can be secured without modifying its configuration.

The machine running the SSH client initiates the tunnel. The SSH server acts as the middleman, forwarding traffic between your client and the destination service. Both ends need SSH access, but the destination service itself doesn't require any changes.

## Local Port Forwarding

Local port forwarding binds a port on your local machine that, when connected to, forwards traffic through the SSH server to a destination. This is useful when the destination service exists on the remote network but isn't directly accessible to you.

The syntax follows:

```bash
ssh -L local_port:destination_host:destination_port user@ssh_server
```

Suppose you have a MySQL database running on `db-server.internal` (IP 192.168.1.100) that's accessible from your SSH server but not from your local machine. Forward local port 3306 to reach it:

```bash
ssh -L 3306:192.168.1.100:3306 user@ssh_server
```

Now connect your MySQL client to `localhost:3306`. The connection travels encrypted to your SSH server, then continues to the database server on the internal network. Your database client doesn't need any special configuration—it simply connects to localhost.

For a web service on an internal server, forward port 8080:

```bash
ssh -L 8080:10.0.0.50:80 user@jump-server
```

Access the internal webapp at `http://localhost:8080`. This pattern works with any TCP service—Redis, PostgreSQL, custom APIs.

## Remote Port Forwarding

Remote port forwarding does the opposite: it makes a local service accessible through the SSH server. This is valuable when you need someone else to access a service on your machine, or when your local machine can't receive incoming connections but can initiate outbound SSH.

The syntax mirrors local forwarding:

```bash
ssh -R remote_port:localhost:local_port user@ssh_server
```

To expose a development server on your laptop to a colleague:

```bash
ssh -R 8080:localhost:3000 user@public-server
```

Your colleague visits `http://public-server:8080`, and the request routes through the SSH server to your local port 3000. This works without opening firewall ports on your end.

A practical use case: running a webhook receiver locally during development. Many services require a public URL for callbacks. Instead of deploying to a server, forward a public port:

```bash
ssh -R 80:localhost:3000 user@tunnel-server
```

Now configure your webhook URL to point to your tunnel server. Traffic arrives at your local development environment.

## Dynamic Port Forwarding

Dynamic port forwarding turns your SSH client into a SOCKS proxy. Unlike local forwarding, which targets a single destination, dynamic forwarding lets you route traffic to any destination through the SSH server. This functions like a minimal VPN.

```bash
ssh -D 1080 user@ssh_server
```

Configure your browser or application to use `localhost:1080` as a SOCKS5 proxy. All connections then tunnel through your SSH server, with the server acting as the exit point.

This approach protects browsing on untrusted networks. Connect to any WiFi, establish the tunnel, and route your traffic through your trusted SSH server. The local network sees only encrypted SSH traffic.

For Chrome, launch with proxy flags:

```bash
google-chrome --proxy-server="socks5://localhost:1080"
```

Firefox has built-in SOCKS proxy settings in Preferences. Command-line tools often accept proxy environment variables:

```bash
export ALL_PROXY="socks5://localhost:1080"
curl https://example.com
```

## Persisting Tunnels

SSH tunnels close when the SSH session ends. For persistent tunnels, use autossh or systemd:

```bash
autossh -M 20000 -f -N -L 3306:192.168.1.100:3306 user@ssh_server
```

The `-M 20000` flag monitors the tunnel on port 20000, reconnecting automatically if it drops. The `-f` backgrounds the process.

On systems with systemd, create a service file at `~/.config/systemd/user/ssh-tunnel.service`:

```ini
[Unit]
Description=SSH Tunnel to db-server

[Service]
Type=simple
ExecStart=/usr/bin/ssh -N -L 3306:192.168.1.100:3306 user@ssh_server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Enable with:

```bash
systemctl --user enable ssh-tunnel.service
systemctl --user start ssh-tunnel.service
```

The tunnel now survives disconnections and survives system restarts.

## Security Considerations

SSH tunnels inherit SSH's security properties. Use key-based authentication rather than passwords. Generate ed25519 keys for modern systems:

```bash
ssh-keygen -t ed25519 -C "tunnel-key"
```

Add the public key to your SSH server's `authorized_keys` file. For additional security, restrict the key to specific commands using the `command` option in authorized_keys:

```command="echo 'Port forwarding only'",no-port-forwarding,no-x11-forwarding,no-pty ssh-ed25519 AAAA...
```

This prevents the key from being used for interactive sessions while allowing tunnels.

Avoid forwarding to sensitive services over tunnels if the SSH server itself isn't trusted. The server can observe forwarded traffic, though it cannot decrypt it without compromising the connection endpoints.

## Troubleshooting SSH Tunnels

Common issues and solutions:

**Connection Refused on Local Port**:
```bash
# Port already in use - try a higher number
ssh -L 9306:192.168.1.100:3306 user@ssh_server
# Or kill existing process
lsof -i :3306 | grep -v COMMAND | awk '{print $2}' | xargs kill
```

**Tunnel Works Briefly Then Drops**:
```bash
# Enable connection keep-alive
ssh -o ServerAliveInterval=60 -L 3306:192.168.1.100:3306 user@ssh_server

# Or configure in ~/.ssh/config
Host jump-server
  HostName ssh_server
  User user
  ServerAliveInterval 60
  ServerAliveCountMax 10
```

**SOCKS Proxy Stops Working**:
```bash
# Verify SOCKS is listening
netstat -an | grep 1080

# Test proxy is actually being used
curl -x socks5://localhost:1080 https://example.com -v
# Should show "* SOCKS 5 connect"
```

## Advanced Pattern: Recursive Tunneling

For multi-hop scenarios (access server A through server B through server C):

```bash
# Connection chain: local → server_b → server_a → internal_service
# First establish tunnel from local to server_b
ssh -L 3307:server_a:3306 user@server_b

# Then in another terminal, connect to server_a through first tunnel
ssh -L 3306:internal_server:3306 -p 3307 localhost
# Now localhost:3306 reaches the internal service through 2 hops
```

For complex setups, use SSH ProxyJump instead:

```bash
# Single command equivalent
ssh -J user@server_b user@server_a -L 3306:internal_server:3306
```

## Threat Model: SSH Tunneling Security Assumptions

SSH tunneling provides encryption but has limitations:

**What it protects**:
- Encrypts traffic between tunnel endpoints
- Prevents ISP/network monitoring from seeing what you're accessing
- Prevents man-in-the-middle attacks on the tunnel itself

**What it doesn't protect**:
- The SSH server can see all forwarded traffic (unless encrypted end-to-end)
- DNS queries may leak (outside the tunnel)
- IP addresses of tunnel endpoints are visible to any network observer
- The SSH server can be compromised, exposing all traffic it handles

**Risk scenarios**:
- **Untrusted network + untrusted SSH server**: Traffic is encrypted but SSH server operator has complete visibility
- **Public WiFi + trusted SSH server**: Good protection against WiFi monitoring, but trust server completely
- **Compromised SSH server**: All forwarded traffic is compromised

For high-security scenarios, use VPN or tor instead. For standard privacy protection, SSH tunneling to a trusted server works well.

## Production-Ready SSH Tunnel Wrapper

For real deployments, wrap SSH tunneling in a management script:

```bash
#!/bin/bash
# ssh-tunnel-manager.sh

TUNNEL_NAME="db_tunnel"
TUNNEL_HOST="user@ssh_server"
LOCAL_PORT="3306"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="3306"
PIDFILE="/tmp/${TUNNEL_NAME}.pid"

start_tunnel() {
    echo "Starting SSH tunnel..."
    autossh -M 0 -f -N -L ${LOCAL_PORT}:${REMOTE_HOST}:${REMOTE_PORT} ${TUNNEL_HOST}
    PID=$!
    echo $PID > $PIDFILE
    echo "Tunnel started (PID: $PID)"
}

stop_tunnel() {
    if [ -f $PIDFILE ]; then
        PID=$(cat $PIDFILE)
        kill $PID 2>/dev/null
        rm $PIDFILE
        echo "Tunnel stopped"
    fi
}

status_tunnel() {
    if [ -f $PIDFILE ]; then
        PID=$(cat $PIDFILE)
        if kill -0 $PID 2>/dev/null; then
            echo "Tunnel is running (PID: $PID)"
        else
            echo "Tunnel is not running"
            rm $PIDFILE
        fi
    else
        echo "Tunnel is not running"
    fi
}

case "$1" in
    start) start_tunnel ;;
    stop) stop_tunnel ;;
    restart) stop_tunnel; start_tunnel ;;
    status) status_tunnel ;;
    *) echo "Usage: $0 {start|stop|restart|status}" ;;
esac
```

Usage:
```bash
./ssh-tunnel-manager.sh start
./ssh-tunnel-manager.sh status
./ssh-tunnel-manager.sh stop
```

## Quick Reference

| Tunnel Type | Use Case | Command |
|-------------|----------|---------|
| Local (-L) | Access remote service locally | `ssh -L local:remote_host:remote_port user@server` |
| Remote (-R) | Expose local service remotely | `ssh -R remote_port:localhost:local_port user@server` |
| Dynamic (-D) | SOCKS proxy for all traffic | `ssh -D 1080 user@server` |

SSH tunneling provides encrypted paths between devices without the overhead of full VPN solutions. Local forwarding reaches services on remote networks. Remote forwarding exposes local services externally. Dynamic forwarding creates personal SOCKS proxies. Combine these patterns with persistent connections for reliable infrastructure.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
