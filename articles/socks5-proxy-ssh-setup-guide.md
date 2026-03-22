---
layout: default
title: "How to Set Up a SOCKS5 Proxy with SSH"
description: "Create a SOCKS5 proxy over SSH for traffic routing, application-level proxying, and browser isolation without installing extra VPN software"
date: 2026-03-22
author: theluckystrike
permalink: /socks5-proxy-ssh-setup-guide/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up a SOCKS5 Proxy with SSH

SSH's built-in dynamic port forwarding turns any SSH connection into a SOCKS5 proxy. Traffic you route through it exits at the remote server, not your local machine. Unlike a VPN, it works at the application level — you choose which apps use it — and requires no additional server software beyond sshd.

## Key Takeaways

- **This key can be**: passphrase-free since it runs as a service, but restrict it server-side.
- **The key can only**: be used for forwarding.
- **The basic SOCKS5 proxy via SSH has no username/password layer**: rely on network-level access controls.
- **Unlike a VPN, it works at the application level**: you choose which apps use it — and requires no additional server software beyond sshd.
- **Any app that supports**: SOCKS5 can route traffic through it.

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

## Related Reading

- [How to Use SSH Tunneling for Encrypted Communication](/privacy-tools-guide/how-to-use-ssh-tunneling-for-encrypted-communication-between/)
- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
