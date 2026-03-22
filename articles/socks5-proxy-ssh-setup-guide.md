---
layout: default
title: "How to Set Up a SOCKS5 Proxy with SSH"
description: "Create a SOCKS5 proxy over SSH for traffic routing, application-level proxying, and browser isolation without installing extra VPN software"
date: 2026-03-22
author: theluckystrike
permalink: /socks5-proxy-ssh-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up a SOCKS5 Proxy with SSH

SSH's built-in dynamic port forwarding turns any SSH connection into a SOCKS5 proxy. Traffic you route through it exits at the remote server, not your local machine. Unlike a VPN, it works at the application level — you choose which apps use it — and requires no additional server software beyond sshd.

## Table of Contents

- [Basic SOCKS5 Proxy in One Command](#basic-socks5-proxy-in-one-command)
- [Configure Applications to Use the Proxy](#configure-applications-to-use-the-proxy)
- [Persistent Connection with systemd](#persistent-connection-with-systemd)
- [Restricting the Server-Side Key](#restricting-the-server-side-key)
- [Bind to a Specific Interface](#bind-to-a-specific-interface)
- [Multiple Hops (Jump Hosts)](#multiple-hops-jump-hosts)
- [DNS Leak Check](#dns-leak-check)
- [Comparing SSH SOCKS5 vs Other Options](#comparing-ssh-socks5-vs-other-options)
- [Advanced SSH Configuration for SOCKS Proxies](#advanced-ssh-configuration-for-socks-proxies)
- [Monitoring Proxy Health](#monitoring-proxy-health)
- [Debugging Connection Issues](#debugging-connection-issues)
- [Application-Level Configuration Examples](#application-level-configuration-examples)
- [Performance Optimization](#performance-optimization)
- [Threat Model: SOCKS vs VPN vs Tor](#threat-model-socks-vs-vpn-vs-tor)
- [Security Hardening for Production](#security-hardening-for-production)
- [Troubleshooting DNS Leaks with SOCKS5](#troubleshooting-dns-leaks-with-socks5)
- [Related Reading](#related-reading)

## Basic SOCKS5 Proxy in One Command

```bash
# -D: dynamic port forwarding (SOCKS5 on local port 1080)
# -N: don't execute a remote command
# -f: fork to background after authentication
# -C: compress traffic

ssh -D 1080 -N -f -C user@your-server.example.com

# Verify the proxy is listening
ss -tlnp | grep 1080
# or
netstat -tlnp | grep 1080
```

The proxy now listens on `127.0.0.1:1080`. Any app that supports SOCKS5 can route traffic through it.

## Configure Applications to Use the Proxy

### Firefox

1. Settings → Network Settings → Manual proxy configuration
2. SOCKS Host: `127.0.0.1`, Port: `1080`
3. Select **SOCKS v5**
4. Check **Proxy DNS when using SOCKS v5** — this prevents DNS leaks

### curl

```bash
# Use the proxy for a single request
curl --socks5-hostname 127.0.0.1:1080 https://ifconfig.me

# Set as default in ~/.curlrc
echo 'socks5-hostname = 127.0.0.1:1080' >> ~/.curlrc

# Verify your exit IP matches the server
curl --socks5-hostname 127.0.0.1:1080 https://api.ipify.org
```

### System-wide on Linux (environment variables)

```bash
# These are respected by most command-line tools
export ALL_PROXY=socks5h://127.0.0.1:1080
export HTTPS_PROXY=socks5h://127.0.0.1:1080
export HTTP_PROXY=socks5h://127.0.0.1:1080
# socks5h means the hostname resolution also happens through the proxy
```

### Git

```bash
git config --global http.proxy socks5h://127.0.0.1:1080
git config --global https.proxy socks5h://127.0.0.1:1080

# Per-repo only
git config http.proxy socks5h://127.0.0.1:1080
```

### Python requests

```python
import requests

proxies = {
    "http": "socks5h://127.0.0.1:1080",
    "https": "socks5h://127.0.0.1:1080",
}

# Requires: pip install requests[socks]
response = requests.get("https://api.ipify.org", proxies=proxies)
print(response.text)  # Should show your server's IP
```

## Persistent Connection with systemd

For a proxy that survives reboots and reconnects automatically:

```ini
# /etc/systemd/system/ssh-socks-proxy.service
[Unit]
Description=SSH SOCKS5 Proxy
After=network.target
Wants=network.target

[Service]
Type=simple
User=yourlocaluser
ExecStart=/usr/bin/ssh \
  -D 1080 \
  -N \
  -o ServerAliveInterval=60 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes \
  -o StrictHostKeyChecking=yes \
  -i /home/yourlocaluser/.ssh/proxy_key \
  user@your-server.example.com
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now ssh-socks-proxy

# Check status
sudo systemctl status ssh-socks-proxy
journalctl -u ssh-socks-proxy -f
```

Use a dedicated SSH key (`proxy_key`) for the proxy — separate from your login key. This key can be passphrase-free since it runs as a service, but restrict it server-side.

## Restricting the Server-Side Key

On the remote server, limit what the proxy key can do:

```
# ~user/.ssh/authorized_keys on the server
# Prepend these options before the public key:
no-pty,no-X11-forwarding,no-agent-forwarding,no-user-rc,permitopen="*:*" ssh-ed25519 AAAA... proxy-key
```

The `no-pty` option prevents interactive shell access using this key. The key can only be used for forwarding.

## Bind to a Specific Interface

By default, `-D 1080` binds to `127.0.0.1` — local only. To share the proxy with other machines on your LAN:

```bash
# Bind to all interfaces (careful — firewalled network only)
ssh -D 0.0.0.0:1080 -N -f user@server

# Bind to LAN IP specifically
ssh -D 192.168.1.100:1080 -N -f user@server
```

If you share the proxy on your network, require authentication. The basic SOCKS5 proxy via SSH has no username/password layer — rely on network-level access controls.

## Multiple Hops (Jump Hosts)

Route through an intermediate host before reaching the proxy server:

```bash
# Proxy via jump host
ssh -J jumpuser@jump.example.com -D 1080 -N -f proxyuser@destination.example.com

# Or in ~/.ssh/config
Host proxy-via-jump
  HostName destination.example.com
  User proxyuser
  ProxyJump jumpuser@jump.example.com
  DynamicForward 1080
  IdentityFile ~/.ssh/proxy_key
```

```bash
# Then simply
ssh -Nf proxy-via-jump
```

## DNS Leak Check

Confirm DNS queries go through the proxy, not your local resolver:

```bash
# With proxy active, check DNS from a remote perspective
curl --socks5-hostname 127.0.0.1:1080 https://1.1.1.1/cdn-cgi/trace | grep ip

# Check if DNS is leaking locally
# Run tcpdump locally while making a request through the proxy
sudo tcpdump -i any -n port 53 &
curl --socks5-hostname 127.0.0.1:1080 https://example.com
# If you see DNS queries in tcpdump, use socks5h:// (hostname resolution through proxy)
```

## Comparing SSH SOCKS5 vs Other Options

| Method | Setup | Speed | Per-app | DNS leak protection |
|--------|-------|-------|---------|---------------------|
| SSH SOCKS5 | Minutes | Fast | Yes | With socks5h |
| WireGuard VPN | More setup | Very fast | No (system-wide) | Built-in |
| Tor | Minutes | Slow | Via browser/torsocks | Built-in |
| HTTP proxy | Simple | Fast | Yes | No |

SSH SOCKS5 is the right choice when you already have an SSH server, need app-level proxying, and do not want to install VPN software.

## Advanced SSH Configuration for SOCKS Proxies

For power users managing multiple proxies, ~/.ssh/config simplifies management:

```
# ~/.ssh/config for multiple SOCKS proxies

Host proxy-us
  HostName us-server.example.com
  User proxyuser
  IdentityFile ~/.ssh/proxy_us_key
  DynamicForward 127.0.0.1:1080
  ServerAliveInterval 60
  ServerAliveCountMax 3
  ExitOnForwardFailure yes
  StrictHostKeyChecking accept-new

Host proxy-eu
  HostName eu-server.example.com
  User proxyuser
  IdentityFile ~/.ssh/proxy_eu_key
  DynamicForward 127.0.0.1:1081
  ServerAliveInterval 60
  ServerAliveCountMax 3

Host proxy-asia
  HostName asia-server.example.com
  User proxyuser
  IdentityFile ~/.ssh/proxy_asia_key
  DynamicForward 127.0.0.1:1082
  ProxyJump jumphost.example.com
```

Connect using:
```bash
# Opens SOCKS proxy on :1080
ssh -N proxy-us

# Open proxy-eu on different port
ssh -N proxy-eu
```

## Monitoring Proxy Health

Ensure your proxy remains connected and responsive:

```bash
#!/bin/bash
# Monitor SOCKS proxy health

PROXY_PORT=1080
CHECK_INTERVAL=30

while true; do
  # Test connectivity through proxy
  response=$(curl -m 5 --socks5-hostname 127.0.0.1:$PROXY_PORT https://api.ipify.org 2>/dev/null)

  if [ $? -eq 0 ]; then
    echo "Proxy healthy: $response"
  else
    echo "Proxy failed at $(date)"
    # Reconnect
    ssh -N -D $PROXY_PORT user@server &
  fi

  sleep $CHECK_INTERVAL
done
```

## Debugging Connection Issues

When the proxy fails to connect, these debugging steps help:

```bash
# 1. Verify SSH connection works
ssh -v user@server 'echo "SSH OK"'

# 2. Check if proxy port is listening
ss -tlnp | grep 1080

# 3. Test DNS through proxy
curl --socks5-hostname 127.0.0.1:1080 https://1.1.1.1/cdn-cgi/trace

# 4. Verify traffic goes through proxy with tcpdump
sudo tcpdump -i any 'dst port 1080' -A

# 5. Check for IPv6 leaks (if proxy is IPv6-capable)
curl -6 --socks5-hostname 127.0.0.1:1080 https://ifconfig.co/json

# 6. Test with specific ciphers if connection fails
ssh -c aes256-gcm@openssh.com -D 1080 user@server
```

## Application-Level Configuration Examples

### Rust/Cargo

```bash
# ~/.cargo/config.toml
[net]
git-fetch-with-cli = true

# Set environment variables
export ALL_PROXY=socks5h://127.0.0.1:1080
cargo fetch
```

### Node.js/npm

```bash
# ~/.npmrc
https-proxy=socks5h://127.0.0.1:1080
proxy=socks5h://127.0.0.1:1080

# Or via command line
npm install --https-proxy socks5h://127.0.0.1:1080
```

### Docker

```bash
# Configure Docker daemon to use SOCKS proxy
# /etc/docker/daemon.json
{
  "proxies": {
    "https-proxy": "socks5h://127.0.0.1:1080"
  }
}

sudo systemctl restart docker
docker pull alpine  # Will route through proxy
```

### Java Applications

```bash
# java -D flags for SOCKS proxy
java -DsocksProxyHost=127.0.0.1 \
     -DsocksProxyPort=1080 \
     -DsocksProxyVersion=5 \
     -jar application.jar
```

## Performance Optimization

SOCKS proxies can experience latency. Optimize for your use case:

```bash
# Test proxy latency
for i in {1..5}; do
  time curl -s --socks5-hostname 127.0.0.1:1080 https://example.com > /dev/null
done

# Monitor connection quality
watch -n 1 'curl -s --socks5-hostname 127.0.0.1:1080 https://api.ipify.org'
```

Typical latencies:
- Same region (US to US): 5-30ms
- Adjacent region (US to EU): 80-150ms
- Distant region (US to Asia): 150-300ms
- High-latency links: 400ms+

For real-time applications (gaming, video), 200ms+ becomes noticeable.

## Threat Model: SOCKS vs VPN vs Tor

Choose the right solution for your threat model:

| Threat | SOCKS5 SSH | VPN | Tor |
|--------|-----------|-----|-----|
| ISP sees destination | No | No | No |
| Server sees your IP | No | No | No |
| Exit node logging | Depends on host | Depends on provider | No (distributed) |
| Latency | Low | Low | High |
| Bandwidth restriction | None | Often limited | Low |
| Setup complexity | Medium | Low | Low |
| Cost | Server cost | $5-15/mo | Free |
| App-level control | Yes | No | Browser only |

**Choose SSH SOCKS5 if**: You control both endpoints or trust your proxy host, need low latency, and want app-level control.

**Choose VPN if**: You need "set and forget" system-wide encryption without managing infrastructure.

**Choose Tor if**: You need maximum anonymity and don't mind latency overhead.

## Security Hardening for Production

If you're running a proxy server for others:

```bash
# Restrict SSH key to SOCKS forwarding only
# /root/.ssh/authorized_keys (server-side)
restrict,pty,command="/bin/false" ssh-rsa AAAA...

# Lock down firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp from 203.0.113.0/24  # Your IP only
ufw enable

# Monitor proxy usage
netstat -an | grep ESTABLISHED | grep 127.0.0.1:1080 | wc -l

# Log all connections
socat - TCP-LISTEN:1080 | tee -a /var/log/socks-proxy.log
```

## Troubleshooting DNS Leaks with SOCKS5

DNS can leak your real location even through a proxy. Test thoroughly:

```bash
# Test for DNS leaks
curl --socks5-hostname 127.0.0.1:1080 https://dnsleaktest.com/api/v1/my/nameservers

# Compare with direct connection
curl https://dnsleaktest.com/api/v1/my/nameservers

# They should show different DNS providers

# If leaking, use socks5h (h = hostname resolution through proxy)
curl -v --socks5h 127.0.0.1:1080 https://example.com
```

## Related Reading

- [How to Use SSH Tunneling for Encrypted Communication](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
