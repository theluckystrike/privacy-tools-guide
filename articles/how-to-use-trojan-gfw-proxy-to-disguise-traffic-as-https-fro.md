---
layout: default
title: "How To Use Trojan Gfw Proxy To Disguise Traffic As Https Fro"
description: "A practical guide for developers and power users on configuring Trojan GFW proxy to mask traffic as HTTPS for use from China"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-use-trojan-gfw-proxy-to-disguise-traffic-as-https-fro/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Trojan GFW proxy represents a sophisticated approach to network traffic obfuscation, designed specifically to bypass the Great Firewall of China (GFW). Unlike traditional VPN protocols that rely on known signatures, Trojan imitates regular HTTPS traffic, making it significantly harder for deep packet inspection (DPI) systems to detect and block. This guide walks you through the complete setup process, from server configuration to client installation, with practical code examples you can implement immediately.

## Understanding How Trojan Works

Trojan operates on a deceptively simple principle: it wraps encrypted traffic in the TLS protocol, the same encryption used by every secure website on the internet. When you connect to a Trojan server, your traffic appears identical to a normal HTTPS connection to any web server. The GFW cannot distinguish between legitimate HTTPS traffic and Trojan-encrypted data because both use the same port (443) and identical TLS handshakes.

The protocol achieves this by acting as a reverse proxy. Your client connects to the Trojan server on port 443, sends a pre-shared password as the first message, and if authenticated, the server forwards your traffic to its actual destination. This design means that from an external perspective, your connection looks exactly like a standard TLS connection to a web server.

## Server-Side Configuration

Setting up a Trojan server requires a Linux VPS with TLS certificate support. You'll need a domain name pointing to your server and root or sudo access.

First, install the Trojan binary on your server:

```bash
# Download and install Trojan
wget https://github.com/trojan-gfw/trojan/releases/download/v1.16.0/trojan-1.16.0-linux-amd64.tar.xz
tar -xf trojan-1.16.0-linux-amd64.tar.xz
sudo cp trojan /usr/local/bin/trojan
sudo chmod +x /usr/local/bin/trojan
```

Obtain a TLS certificate using Certbot:

```bash
sudo apt update && sudo apt install certbot python3-certbot-nginx -y
sudo certbot certonly --standalone -d your-domain.com
```

Create the Trojan server configuration file at `/etc/trojan/config.json`:

```json
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 443,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "password": [
        "your-secure-password-here"
    ],
    "tls": {
        "cert": "/etc/letsencrypt/live/your-domain.com/fullchain.pem",
        "key": "/etc/letsencrypt/live/your-domain.com/privkey.pem",
        "sni": "your-domain.com",
        "alpn": [
            "http/1.1"
        ],
        "prefer_server_cipher": true
    },
    "tcp": {
        "fast_open": true,
        "fast_open_qlen": 256
    },
    "log_level": 1,
    "log_file": "/var/log/trojan.log"
}
```

Replace `your-secure-password-here` with a strong password (use a password manager to generate 32+ random characters). Start the Trojan service:

```bash
sudo systemctl create-unit-file --name trojan.service --force --stdin << EOF
[Unit]
Description=Trojan GFW Proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/trojan /etc/trojan/config.json
Restart=always
User=nobody

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now trojan
```

## Client-Side Configuration

On the client side, you have multiple options depending on your operating system. For desktop systems, the Qt-based Trojan-Qt5 or Trojan-GUI clients provide graphical interfaces. For mobile, the ignoring Android client or ShadowRocket on iOS work well.

For developers who prefer command-line tools or need to integrate Trojan into scripts, use the trojan-go client. Install it on Linux/macOS:

```bash
# Install trojan-go
go install github.com/p4gefau1t/trojan-go@latest
sudo mv ~/go/bin/trojan-go /usr/local/bin/trojan-go
```

Create a client configuration file:

```json
{
    "run_type": "client",
    "local_addr": "127.0.0.1",
    "local_port": 1080,
    "remote_addr": "your-domain.com",
    "remote_port": 443,
    "password": "your-secure-password-here",
    "tls": {
        "sni": "your-domain.com",
        "alpn": ["http/1.1"],
        "verify": true,
        "verify_hostname": true,
        "cert": ""
    },
    "tcp": {
        "fast_open": true,
        "fast_open_qlen": 256
    }
}
```

Start the client in SOCKS5 mode:

```bash
trojan-go -config client-config.json
```

Your local SOCKS5 proxy now runs at `127.0.0.1:1080`.

## Integrating with Your Workflow

For browser-based browsing, configure your system or browser to use the SOCKS5 proxy. For command-line tools, use `proxychains` or set environment variables:

```bash
# Using proxychains with any CLI tool
export PROXY_SOCKS5="127.0.0.1:1080"
proxychains curl https://target-website.com

# Direct environment variable usage
export http_proxy="socks5://127.0.0.1:1080"
export https_proxy="socks5://127.0.0.1:1080"
curl https://target-website.com
```

For Docker containers, configure the daemon or individual containers to route through the proxy by adding environment variables in your Dockerfile or docker-compose.yml.

## Performance Optimization

Trojan's performance depends heavily on your server's network quality and TLS configuration. Enable TCP Fast Open on both client and server (kernel parameters must support it):

```bash
# Add to /etc/sysctl.conf
net.ipv4.tcp_fastopen = 3

# Apply changes
sudo sysctl -p
```

For production deployments, consider running Trojan behind Nginx with WebSocket support for better traffic blending. This allows your server to serve legitimate HTTPS content while simultaneously handling Trojan connections, making traffic analysis even more difficult.

## Troubleshooting Common Issues

If connections fail, verify your firewall rules allow outbound traffic on port 443. Check that your TLS certificate is valid and not expired:

```bash
sudo certbot renew --dry-run
```

For connection timeouts, ensure your server's TCP BBR congestion control is enabled:

```bash
sudo sysctl net.ipv4.tcp_congestion_control=bbr
sudo modprobe bbr
```

Monitor logs on both client and server to identify authentication failures or network issues. The Trojan protocol is designed to be silent on failure—it simply drops invalid packets rather than sending rejection messages.

## Security Considerations

Trojan provides strong traffic obfuscation but remember that no tool guarantees complete invisibility. Advanced traffic analysis can still identify Trojan connections through timing patterns and packet sizes. For maximum security, consider routing all traffic through your Trojan server rather than split-tunneling, and regularly rotate your passwords.

The implementation described here gives you a functional, privacy-respecting proxy suitable for personal use and development work. With proper TLS certificates and secure password practices, your traffic remains encrypted and indistinguishable from normal HTTPS connections.




## Related Reading

- [How To Set Up V2ray Vmess For Accessing Blocked Websites Fro](/privacy-tools-guide/how-to-set-up-v2ray-vmess-for-accessing-blocked-websites-fro/)
- [Cname Cloaking How Trackers Disguise As First Party Dns Expl](/privacy-tools-guide/cname-cloaking-how-trackers-disguise-as-first-party-dns-expl/)
- [How To Use Naiveproxy To Disguise Censorship Circumvention T](/privacy-tools-guide/how-to-use-naiveproxy-to-disguise-censorship-circumvention-t/)
- [Configure Xray Reality Protocol for Undetectable Proxy from Censored Countries](/privacy-tools-guide/how-to-configure-xray-reality-protocol-for-undetectable-prox/)
- [How To Set Up Dnscrypt Proxy For Authenticated Encrypted Dns](/privacy-tools-guide/how-to-set-up-dnscrypt-proxy-for-authenticated-encrypted-dns/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
