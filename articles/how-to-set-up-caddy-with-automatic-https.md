---
layout: default
title: "How to Set Up Caddy with Automatic HTTPS"
description: "Deploy Caddy web server with automatic TLS certificate provisioning, reverse proxying, and security headers for self-hosted services"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-caddy-with-automatic-https/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Caddy is a web server written in Go that automatically provisions and renews TLS certificates from Let's Encrypt. There is no certbot, no cron jobs, no manual renewal. Caddy handles it all. This makes it the fastest way to get a self-hosted service behind HTTPS on a VPS.

Prerequisites

- A VPS with a public IP
- A domain pointing at your VPS (A record)
- Ports 80 and 443 open (Caddy needs both for ACME HTTP-01 challenge)
- Ubuntu 20.04 or later

Step 1 - Install Caddy

```bash
Install from the official Caddy repository
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | \
  sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update && sudo apt install caddy

Verify
caddy version
v2.8.x
```

Caddy installs as a systemd service and starts listening on port 80 immediately. It serves a placeholder page until you configure your Caddyfile.

Step 2 - Basic Caddyfile

The Caddyfile lives at `/etc/caddy/Caddyfile`. Edit it:

```bash
sudo nano /etc/caddy/Caddyfile
```

Minimal configuration to serve a static site with automatic HTTPS:

```
example.com {
    root * /var/www/html
    file_server
}
```

That is the entire configuration. Caddy:
1. Detects that `example.com` is a real domain
2. Provisions a Let's Encrypt certificate automatically
3. Serves files from `/var/www/html`
4. Redirects HTTP to HTTPS automatically

Apply the configuration:
```bash
sudo systemctl reload caddy
sudo systemctl status caddy
```

Check certificate:
```bash
curl -vI https://example.com 2>&1 | grep -A2 "SSL certificate\|subject:\|issuer:"
```

Step 3 - Reverse Proxy to a Backend

The most common use case. proxying to an application running on localhost:

```
example.com {
    reverse_proxy localhost:8080
}

api.example.com {
    reverse_proxy localhost:3000
}
```

Caddy forwards all headers and handles TLS termination. The backend only needs to listen on a local port.

For multiple services on subdomains:

```
Wildcard cert (requires DNS challenge. see below)
*.example.com {
    @wiki host wiki.example.com
    @gitea host gitea.example.com
    @grafana host grafana.example.com

    handle @wiki {
        reverse_proxy localhost:3001
    }
    handle @gitea {
        reverse_proxy localhost:3000
    }
    handle @grafana {
        reverse_proxy localhost:3002
    }
}
```

Step 4 - Security Headers

Add security headers globally using the `header` directive:

```
(security_headers) {
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        X-XSS-Protection "1; mode=block"
        Referrer-Policy "strict-origin-when-cross-origin"
        Permissions-Policy "geolocation=(), microphone=(), camera=()"
        Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
        -Server
        -X-Powered-By
    }
}

example.com {
    import security_headers
    reverse_proxy localhost:8080
}
```

The `-Server` and `-X-Powered-By` lines remove those response headers entirely to reduce fingerprinting.

Step 5 - Rate Limiting

Caddy's rate limit module is not built-in but available as a plugin. For basic protection, use the `respond` directive to block excessive requests, or deploy the `caddy-ratelimit` plugin:

```bash
Build caddy with rate limit module
xcaddy build --with github.com/mholt/caddy-ratelimit

sudo mv caddy /usr/bin/caddy
sudo systemctl restart caddy
```

Then in your Caddyfile:

```
example.com {
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }
    reverse_proxy localhost:8080
}
```

Step 6 - Basic Authentication

Protect an internal service with an username/password:

```bash
Generate a bcrypt-hashed password
caddy hash-password --plaintext 'your-strong-password'
$2a$14$...
```

```
private.example.com {
    basicauth {
        alice $2a$14$...hashedpassword...
    }
    reverse_proxy localhost:8081
}
```

Step 7 - Logging

Enable structured JSON access logs for security analysis:

```
example.com {
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 5
        }
        format json
        level INFO
    }
    reverse_proxy localhost:8080
}
```

Create the log directory:
```bash
sudo mkdir -p /var/log/caddy
sudo chown caddy:caddy /var/log/caddy
```

Parse logs:
```bash
Top 10 IPs by request count
cat /var/log/caddy/access.log | jq -r '.request.remote_ip' | sort | uniq -c | sort -rn | head

Show 404s
cat /var/log/caddy/access.log | jq 'select(.status == 404) | {uri: .request.uri, ip: .request.remote_ip}'
```

Step 8 - DNS Challenge for Wildcard Certificates

Wildcard certs (`*.example.com`) require DNS-01 challenge, which means Caddy needs API access to your DNS provider. Install the appropriate DNS module:

```bash
For Cloudflare
xcaddy build --with github.com/caddy-dns/cloudflare

In Caddyfile, provide the API token
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
    # ... your handlers
}
```

Set the environment variable:
```bash
/etc/systemd/system/caddy.service.d/override.conf
[Service]
Environment="CF_API_TOKEN=your_cloudflare_api_token"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart caddy
```

Verify Everything

```bash
Check certificate status
sudo caddy certificates

Validate your Caddyfile without restarting
caddy validate --config /etc/caddy/Caddyfile

Test TLS configuration
curl -vI https://example.com

Grade your TLS at Qualys SSL Labs
https://www.ssllabs.com/ssltest/analyze.html?d=example.com
```

Related Reading

- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)
- [Secure WebSocket Connections Setup Guide](/secure-websocket-connections-setup-guide/)
- [How to Set Up Authentik for Identity Management](/how-to-set-up-authentik-for-identity-management/)

- [How To Set Up Automatic Account Deletion Triggers If You](/how-to-set-up-automatic-account-deletion-triggers-if-you-bec/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

Related Articles

- [How to Set Up Mullvad VPN on Linux](/mullvad-vpn-linux-setup-guide/)
- [Encrypted DNS over HTTPS on Linux](/encrypted-dns-over-https-linux-setup)
- [Setting Up Vault for Secrets Management](/hashicorp-vault-secrets-management-setup/)
- [How To Use Trojan Gfw Proxy To Disguise Traffic As Https](/how-to-use-trojan-gfw-proxy-to-disguise-traffic-as-https-fro/)
- [How to Set Up Authentik for Identity Management](/how-to-set-up-authentik-for-identity-management/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
