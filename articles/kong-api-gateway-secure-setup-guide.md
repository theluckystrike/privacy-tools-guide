---
layout: default
title: "Secure API Gateway Setup with Kong"
description: "Deploy Kong API Gateway with rate limiting, JWT auth, IP restriction, and TLS termination to protect backend services from abuse and unauthorized access"
date: 2026-03-22
author: theluckystrike
permalink: /kong-api-gateway-secure-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# Secure API Gateway Setup with Kong

Kong is an open-source API gateway that sits in front of your services and enforces authentication, rate limiting, logging, and TLS — without touching application code. This guide covers a production-grade setup using Kong Gateway (OSS) 3.x on Ubuntu 24.04 with PostgreSQL, deployed behind Nginx for TLS termination.

## Architecture Overview

```
Client → Nginx (TLS) → Kong (8000) → Upstream Services
                   ↓
            Kong Admin (8001, localhost-only)
```

Kong handles: JWT validation, rate limiting, IP allowlisting, request logging. Nginx handles: certificate renewal, HTTPS, and hiding Kong's admin port.

---

## 1. Install Kong and PostgreSQL

```bash
# Add Kong repository
curl -fsSL https://packages.konghq.com/public/gateway-38/gpg.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/kong-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/kong-archive-keyring.gpg] \
  https://packages.konghq.com/public/gateway-38/deb/ubuntu \
  $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/kong.list

sudo apt update && sudo apt install -y kong-enterprise-edition || \
sudo apt install -y kong

# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib
```

---

## 2. Configure PostgreSQL

```bash
sudo -u postgres psql <<'SQL'
CREATE USER kong WITH PASSWORD 'StrongPassHere42!';
CREATE DATABASE kong OWNER kong;
SQL
```

---

## 3. Configure Kong

```bash
sudo cp /etc/kong/kong.conf.default /etc/kong/kong.conf
sudo tee /etc/kong/kong.conf > /dev/null <<'EOF'
# Database
database = postgres
pg_host = 127.0.0.1
pg_port = 5432
pg_user = kong
pg_password = StrongPassHere42!
pg_database = kong

# Proxy (public)
proxy_listen = 0.0.0.0:8000
# Admin (local only — NEVER expose to internet)
admin_listen = 127.0.0.1:8001

# TLS — Kong itself can do TLS, but we'll use Nginx in front
proxy_ssl = off

# Log level
log_level = warn
EOF

# Initialize database
sudo kong migrations bootstrap -c /etc/kong/kong.conf

# Start Kong
sudo systemctl enable --now kong
sudo systemctl status kong
```

Test Kong is running:

```bash
curl -s http://127.0.0.1:8001/ | python3 -m json.tool | grep version
```

---

## 4. Add a Service and Route

Services represent upstream APIs. Routes define what paths/hosts map to a service.

```bash
ADMIN="http://127.0.0.1:8001"

# Create a service pointing to your backend
curl -s -X POST $ADMIN/services \
  -d name=my-api \
  -d url=http://127.0.0.1:3000

# Create a route for that service
curl -s -X POST $ADMIN/services/my-api/routes \
  -d 'paths[]=/api' \
  -d strip_path=false

# Verify
curl -s $ADMIN/services | python3 -m json.tool
```

---

## 5. Enable Rate Limiting

Protect backend services from brute-force and DoS:

```bash
# Global rate limiting: 100 requests per minute per IP
curl -s -X POST $ADMIN/plugins \
  -d name=rate-limiting \
  -d config.minute=100 \
  -d config.policy=local \
  -d config.hide_client_headers=false

# Per-service rate limiting: stricter limit for sensitive endpoint
curl -s -X POST $ADMIN/services/my-api/plugins \
  -d name=rate-limiting \
  -d config.minute=30 \
  -d config.hour=500 \
  -d config.policy=redis \
  -d config.redis_host=127.0.0.1 \
  -d config.redis_port=6379
```

Use Redis-backed policy in multi-node deployments to share rate limit state across Kong instances.

---

## 6. Enable JWT Authentication

```bash
# Enable the JWT plugin on your service
curl -s -X POST $ADMIN/services/my-api/plugins \
  -d name=jwt

# Create a consumer
curl -s -X POST $ADMIN/consumers \
  -d username=api-client-1

# Generate a JWT credential
curl -s -X POST $ADMIN/consumers/api-client-1/jwt

# Response includes:
# {
#   "key": "rsa_key_or_hs256_key",
#   "secret": "your_shared_secret",
#   "algorithm": "HS256"
# }
```

Clients must include the JWT in the `Authorization: Bearer <token>` header. Generate tokens in your application:

```python
import jwt, time

payload = {
    "iss": "rsa_key_from_kong",   # must match the 'key' field from Kong
    "sub": "api-client-1",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,
}
token = jwt.encode(payload, "your_shared_secret", algorithm="HS256")
print(token)
```

---

## 7. Enable IP Restriction

Block or allow specific IP ranges:

```bash
# Allow only your office IP and VPN exit node
curl -s -X POST $ADMIN/services/my-api/plugins \
  -d name=ip-restriction \
  -d 'config.allow[]=203.0.113.10' \
  -d 'config.allow[]=10.0.0.0/8'

# Block a known bad actor IP range
curl -s -X POST $ADMIN/plugins \
  -d name=ip-restriction \
  -d 'config.deny[]=198.51.100.0/24'
```

---

## 8. Enable Request Logging

```bash
# Log to file (for local ELK or Splunk ingest)
curl -s -X POST $ADMIN/plugins \
  -d name=file-log \
  -d config.path=/var/log/kong/access.log \
  -d config.reopen=true

# Or HTTP log (forward to logging service)
curl -s -X POST $ADMIN/plugins \
  -d name=http-log \
  -d config.http_endpoint=http://log-aggregator:9200/kong-logs \
  -d config.method=POST \
  -d config.timeout=10000 \
  -d config.keepalive=60000
```

---

## 9. TLS Termination with Nginx

```nginx
# /etc/nginx/sites-available/api-gateway
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_session_cache   shared:SSL:10m;

    # Forward to Kong proxy
    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto https;
        proxy_read_timeout 60s;
    }
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}
```

```bash
sudo certbot --nginx -d api.example.com
sudo nginx -t && sudo systemctl reload nginx
```

---

## 10. Harden Kong Admin

The admin API must never be internet-accessible. Restrict it further:

```bash
# Bind to Unix socket instead of TCP (if single-node)
# In kong.conf:
# admin_listen = unix:/run/kong/kong_admin.sock

# If using TCP, add firewall rule
sudo ufw deny 8001
sudo ufw allow from 127.0.0.1 to any port 8001

# Add basic auth to admin via Nginx proxy (optional)
# Only expose admin behind VPN or bastion host
```

---

## 11. Monitor with Prometheus

```bash
# Enable Prometheus plugin globally
curl -s -X POST $ADMIN/plugins \
  -d name=prometheus

# Kong exposes metrics at :8001/metrics
# Add to prometheus.yml:
# - job_name: kong
#   static_configs:
#     - targets: ['127.0.0.1:8001']
#   metrics_path: /metrics
```

Key metrics: `kong_http_requests_total`, `kong_latency_bucket`, `kong_bandwidth_bytes_total`.

---

## Security Checklist

- Admin API bound to `127.0.0.1` only — never `0.0.0.0`
- Rate limiting enabled globally with tighter per-service limits
- JWT plugin enforced on all authenticated routes
- TLS 1.2+ with strong cipher suites at the Nginx layer
- IP restriction applied to admin and internal routes
- File or HTTP logging forwarded to a SIEM
- Kong version pinned; `apt-mark hold kong` to prevent accidental upgrades

---

## Related Reading

- [Secure JWT Implementation Best Practices](/privacy-tools-guide/secure-jwt-implementation-best-practices/)
- [Secure Webhook Implementation Guide](/privacy-tools-guide/secure-webhook-implementation-guide/)
- [How to Set Up Wazuh SIEM for Small Teams](/privacy-tools-guide/wazuh-siem-small-teams-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
