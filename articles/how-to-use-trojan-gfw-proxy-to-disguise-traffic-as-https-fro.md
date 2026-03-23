---
layout: default
title: "How To Use Trojan Gfw Proxy To Disguise Traffic As Https"
description: "A practical guide for developers and power users on configuring Trojan GFW proxy to mask traffic as HTTPS for use from China"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-use-trojan-gfw-proxy-to-disguise-traffic-as-https-fro/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Trojan GFW proxy represents a sophisticated approach to network traffic obfuscation, designed specifically to bypass the Great Firewall of China (GFW). Unlike traditional VPN protocols that rely on known signatures, Trojan imitates regular HTTPS traffic, making it significantly harder for deep packet inspection (DPI) systems to detect and block. This guide walks you through the complete setup process, from server configuration to client installation, with practical code examples you can implement immediately.

Table of Contents

- [Prerequisites](#prerequisites)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Security Considerations](#security-considerations)

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1: Understand How Trojan Works

Trojan operates on a deceptively simple principle: it wraps encrypted traffic in the TLS protocol, the same encryption used by every secure website on the internet. When you connect to a Trojan server, your traffic appears identical to a normal HTTPS connection to any web server. The GFW cannot distinguish between legitimate HTTPS traffic and Trojan-encrypted data because both use the same port (443) and identical TLS handshakes.

The protocol achieves this by acting as a reverse proxy. Your client connects to the Trojan server on port 443, sends a pre-shared password as the first message, and if authenticated, the server forwards your traffic to its actual destination. This design means that from an external perspective, your connection looks exactly like a standard TLS connection to a web server.

What differentiates Trojan from earlier protocols like Shadowsocks or V2Ray VMess is its use of authentic TLS certificates issued by trusted certificate authorities. When the GFW probes a Trojan server, it receives a valid TLS handshake and then sees what appears to be normal HTTPS responses, because the server also runs a real web server on the fallback port 80. This makes active probing by censors significantly harder than with protocols that use custom encryption schemes.

Step 2: Server-Side Configuration

Setting up a Trojan server requires a Linux VPS with TLS certificate support. You'll need a domain name pointing to your server and root or sudo access.

First, install the Trojan binary on your server:

```bash
Download and install Trojan
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

Step 3: Set Up the Nginx Fallback Server

A critical component of a convincing Trojan deployment is the fallback web server. When the GFW or any censor probes your server without the Trojan password, they should receive a legitimate website response. Run Nginx on port 80 serving real content:

```bash
sudo apt install nginx -y

Create a minimal but convincing site
sudo tee /etc/nginx/sites-available/default << 'EOF'
server {
    listen 80;
    server_name your-domain.com;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

sudo systemctl enable --now nginx
```

The content at `/var/www/html` should be a plausible static site. An empty directory or a default Nginx page looks suspicious under active probing. A minimal blog or landing page with a few HTML files significantly reduces the probability that automated scanning identifies the server as a proxy.

Step 4: Client-Side Configuration

On the client side, you have multiple options depending on your operating system. For desktop systems, the Qt-based Trojan-Qt5 or Trojan-GUI clients provide graphical interfaces. For mobile, the ignoring Android client or ShadowRocket on iOS work well.

For developers who prefer command-line tools or need to integrate Trojan into scripts, use the trojan-go client. Install it on Linux/macOS:

```bash
Install trojan-go
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

Step 5: Integrate with Your Workflow

For browser-based browsing, configure your system or browser to use the SOCKS5 proxy. For command-line tools, use `proxychains` or set environment variables:

```bash
Using proxychains with any CLI tool
export PROXY_SOCKS5="127.0.0.1:1080"
proxychains curl https://target-website.com

Direct environment variable usage
export http_proxy="socks5://127.0.0.1:1080"
export https_proxy="socks5://127.0.0.1:1080"
curl https://target-website.com
```

For Docker containers, configure the daemon or individual containers to route through the proxy by adding environment variables in your Dockerfile or docker-compose.yml.

For browsers, both Firefox and Chrome support SOCKS5 proxy configuration. In Firefox, navigate to Settings > Network Settings and select Manual Proxy Configuration. Set SOCKS Host to `127.0.0.1` and Port to `1080`, select SOCKS v5, and enable "Proxy DNS when using SOCKS v5" to prevent DNS leaks, without this setting, DNS queries bypass the proxy and can reveal your browsing activity to local observers.

Performance Optimization

Trojan's performance depends heavily on your server's network quality and TLS configuration. Enable TCP Fast Open on both client and server (kernel parameters must support it):

```bash
Add to /etc/sysctl.conf
net.ipv4.tcp_fastopen = 3

Apply changes
sudo sysctl -p
```

For production deployments, consider running Trojan behind Nginx with WebSocket support for better traffic blending. This allows your server to serve legitimate HTTPS content while simultaneously handling Trojan connections, making traffic analysis even more difficult.

Enable BBR congestion control on your server for better throughput on high-latency connections, which is typical for cross-border traffic:

```bash
Enable BBR
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sudo sysctl -p

Verify BBR is active
sysctl net.ipv4.tcp_congestion_control
```

BBR significantly improves performance on connections with packet loss, which is common when traffic traverses the GFW. Real-world throughput improvements of 30 to 50 percent are achievable compared to the default CUBIC congestion control algorithm.

Troubleshooting Common Issues

If connections fail, verify your firewall rules allow outbound traffic on port 443. Check that your TLS certificate is valid and not expired:

```bash
sudo certbot renew --dry-run
```

For connection timeouts, ensure your server's TCP BBR congestion control is enabled:

```bash
sudo sysctl net.ipv4.tcp_congestion_control=bbr
sudo modprobe bbr
```

Monitor logs on both client and server to identify authentication failures or network issues. The Trojan protocol is designed to be silent on failure, it simply drops invalid packets rather than sending rejection messages.

If your server is being actively blocked despite correct configuration, the issue may be IP-level blocking rather than DPI detection. Check whether your VPS IP is in commonly blocked ranges using tools like `ping`, `traceroute`, or online GFW checking services. If the IP is blocked at the routing level, switching VPS providers or using CDN fronting through Cloudflare may resolve the issue, configure Cloudflare in front of your domain, set a custom ALPN value, and route Trojan traffic through Cloudflare's 443 port using Websocket mode in trojan-go.

Step 6: Certificate Renewal Automation

TLS certificates issued by Let's Encrypt expire after 90 days. Failing to renew them breaks your proxy silently, the client will reject the expired certificate and connections will fail. Automate renewal with a systemd timer:

```bash
Check that the renewal timer is active
sudo systemctl status certbot.timer

Force a renewal test
sudo certbot renew --dry-run

If Certbot was installed via snap, renewal is automatic
If installed via apt, add a cron job
echo "0 3 * * * root certbot renew --quiet --pre-hook 'systemctl stop trojan' --post-hook 'systemctl start trojan'" | sudo tee /etc/cron.d/certbot-trojan
```

The `--pre-hook` and `--post-hook` options temporarily stop and restart Trojan during renewal, which is necessary if Trojan is binding port 443 and Certbot needs to use the standalone authenticator.

Security Considerations

Trojan provides strong traffic obfuscation but remember that no tool guarantees complete invisibility. Advanced traffic analysis can still identify Trojan connections through timing patterns and packet sizes. For maximum security, consider routing all traffic through your Trojan server rather than split-tunneling, and regularly rotate your passwords.

The implementation described here gives you a functional, privacy-respecting proxy suitable for personal use and development work. With proper TLS certificates and secure password practices, your traffic remains encrypted and indistinguishable from normal HTTPS connections.

Frequently Asked Questions

How long does it take to use trojan gfw proxy to disguise traffic as https?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Best Vpn Protocols That Still Work Inside China After Deep](/best-vpn-protocols-that-still-work-inside-china-after-deep-p/)
- [How To Use Outline Vpn Server For Creating Personal Proxy](/how-to-use-outline-vpn-server-for-creating-personal-proxy-in/)
- [How To Use Naiveproxy To Disguise Censorship Circumvention](/how-to-use-naiveproxy-to-disguise-censorship-circumvention-t/)
- [VPN Traffic Obfuscation Techniques](/vpn-traffic-obfuscation-techniques-shadowsocks-stunnel-compa/)
- [VPN Packet Inspection Explained](/vpn-packet-inspection-how-deep-packet-inspection-detects-vpn-traffic/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
