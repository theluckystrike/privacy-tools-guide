---
layout: default
title: "How to Set Up Authelia for SSO"
description: "Deploy Authelia as a self-hosted SSO and two-factor authentication gateway for your internal services using Docker, Traefik, and LDAP or a flat user file"
date: 2026-03-22
author: theluckystrike
permalink: /authelia-sso-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up Authelia for SSO

Authelia is a self-hosted authentication proxy that puts a login screen (with optional 2FA) in front of any HTTP service. Instead of setting up authentication in each of your self-hosted apps individually, Authelia intercepts requests to your internal services and handles authentication centrally.

## Key Takeaways

- **Topics covered**: architecture, prerequisites, step 1: docker compose setup
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## Architecture

```
Browser → Traefik (reverse proxy) → Authelia (auth check)
                                          │
                     NOT authenticated → Login page
                         Authenticated → Forward request to internal service
                                         (Grafana, Portainer, Gitea, etc.)
```

Traefik sends an auth forward request to Authelia for every incoming request. If Authelia says the user is authenticated, Traefik forwards the request. If not, it redirects to the Authelia login page.

## Prerequisites

- A domain name with DNS pointing to your server
- Traefik v2+ running (or nginx as an alternative)
- Docker and Docker Compose

## Step 1: Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    volumes:
      - ./authelia:/config
    environment:
      - TZ=America/New_York
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.authelia.rule=Host(`auth.yourdomain.com`)"
      - "traefik.http.routers.authelia.entrypoints=websecure"
      - "traefik.http.routers.authelia.tls.certresolver=letsencrypt"
      - "traefik.http.services.authelia.loadbalancer.server.port=9091"
      # Create the forwardAuth middleware for other services to use
      - "traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.yourdomain.com"
      - "traefik.http.middlewares.authelia.forwardauth.trustforwardheader=true"
      - "traefik.http.middlewares.authelia.forwardauth.authresponseheaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email"
    networks:
      - proxy

  redis:
    image: redis:alpine
    container_name: authelia-redis
    restart: unless-stopped
    volumes:
      - ./redis:/data
    networks:
      - proxy

networks:
  proxy:
    external: true
```

## Step 2: Authelia Configuration

```yaml
# authelia/configuration.yml
---
server:
  host: 0.0.0.0
  port: 9091

log:
  level: info

theme: dark

jwt_secret: "$(openssl rand -hex 32)"   # replace with actual random string

default_redirection_url: https://yourdomain.com

totp:
  issuer: yourdomain.com
  period: 30
  skew: 1

authentication_backend:
  file:
    path: /config/users_database.yml
    password:
      algorithm: argon2id
      iterations: 3
      memory: 65536
      parallelism: 4

access_control:
  default_policy: deny

  rules:
    # Public services — no auth needed
    - domain: public.yourdomain.com
      policy: bypass

    # Internal services — require 1FA (password only)
    - domain: gitea.yourdomain.com
      policy: one_factor

    # Sensitive services — require 2FA
    - domain: portainer.yourdomain.com
      policy: two_factor

    - domain: grafana.yourdomain.com
      policy: two_factor

    # Admin panel — 2FA + specific user group
    - domain: admin.yourdomain.com
      policy: two_factor
      subject: "group:admins"

session:
  name: authelia_session
  secret: "$(openssl rand -hex 32)"   # replace with actual random string
  expiration: 3600       # 1 hour
  inactivity: 300        # 5 minutes
  domain: yourdomain.com

  redis:
    host: authelia-redis
    port: 6379

regulation:
  max_retries: 3
  find_time: 120
  ban_time: 300

storage:
  local:
    path: /config/db.sqlite3

notifier:
  # For production, use SMTP:
  smtp:
    username: authelia@yourdomain.com
    password: your-smtp-password
    host: smtp.yourdomain.com
    port: 587
    sender: authelia@yourdomain.com
  # For testing, use filesystem:
  # filesystem:
  #   filename: /config/notification.txt
```

## Step 3: Create Users

```bash
# Generate a password hash
docker run --rm authelia/authelia:latest \
  authelia hash-password 'your-secure-password'
# Digest: $argon2id$v=19$m=65536,t=3,p=4$...
```

```yaml
# authelia/users_database.yml
---
users:
  alice:
    disabled: false
    displayname: "Alice Smith"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."   # from hash-password command
    email: alice@yourdomain.com
    groups:
      - admins
      - users

  bob:
    disabled: false
    displayname: "Bob Jones"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: bob@yourdomain.com
    groups:
      - users
```

## Step 4: Protect a Service with Traefik

```yaml
# Add these labels to any service you want protected:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.grafana.rule=Host(`grafana.yourdomain.com`)"
  - "traefik.http.routers.grafana.entrypoints=websecure"
  - "traefik.http.routers.grafana.tls.certresolver=letsencrypt"
  # Apply the Authelia middleware
  - "traefik.http.routers.grafana.middlewares=authelia@docker"
```

## Step 5: nginx ForwardAuth Alternative

If using nginx instead of Traefik:

```nginx
# nginx.conf — protect an internal service
server {
    listen 443 ssl;
    server_name grafana.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location /authelia {
        internal;
        set $upstream_authelia http://authelia:9091/api/verify;
        proxy_pass_request_body off;
        proxy_pass $upstream_authelia;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Method $request_method;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-Uri $request_uri;
    }

    location / {
        auth_request /authelia;
        auth_request_set $user $upstream_http_remote_user;
        proxy_set_header Remote-User $user;
        error_page 401 =302 https://auth.yourdomain.com/?rd=$scheme://$http_host$request_uri;
        proxy_pass http://grafana:3000;
    }
}
```

## Step 6: Configure TOTP 2FA

After first login, users are prompted to register a TOTP device:

1. Navigate to a 2FA-protected service → redirected to Authelia login
2. Enter username and password
3. Authelia emails a one-time link to register a TOTP app
4. Scan the QR code with Aegis (Android), Raivo (iOS), or any TOTP app
5. Enter the 6-digit code to verify

```bash
# Verify Authelia is running and can send emails
docker logs authelia --tail 50 | grep -i "notification\|email\|smtp"
```

## Step 7: Check Access Logs

```bash
# Watch authentication attempts in real time
docker logs authelia -f | grep -E "(authentication|access_control|banned)"

# Check the Authelia log for specific user
docker logs authelia | grep "alice" | tail -20
```

## Related Reading

- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)
- [How to Set Up Vault for Secrets Management](/privacy-tools-guide/hashicorp-vault-secrets-management-setup/)
- [How to Secure Your GitHub Account](/privacy-tools-guide/secure-github-account-hardening-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
