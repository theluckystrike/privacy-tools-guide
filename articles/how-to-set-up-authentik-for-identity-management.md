---
layout: default
title: "How to Set Up Authentik for Identity Management"
description: "Deploy Authentik as a self-hosted SSO and identity provider with OIDC, SAML, and LDAP for authenticating your self-hosted services"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-authentik-for-identity-management/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Authentik is an open-source identity provider that replaces services like Auth0 or Okta for self-hosted setups. It supports OIDC, SAML, LDAP, and proxy authentication, letting you add single sign-on to any self-hosted service without creating per-service user databases. This guide covers Docker Compose deployment, creating an OIDC provider, and protecting a service with the proxy outpost.

## Prerequisites

- A server with Docker and Docker Compose installed
- A domain with DNS pointing to your server (e.g., `auth.example.com`)
- Ports 80 and 443 available (for the reverse proxy)
- At least 2GB RAM

## Step 1: Deploy Authentik with Docker Compose

Download the official compose file:

```bash
mkdir -p /opt/authentik && cd /opt/authentik

curl -o docker-compose.yml \
  https://goauthentik.io/docker-compose.yml

# Generate a secret key and password
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 36)" > .env
echo "AUTHENTIK_POSTGRESQL__PASSWORD=$(openssl rand -base64 20)" >> .env
echo "PG_PASS=$(cat .env | grep POSTGRESQL__PASSWORD | cut -d= -f2)" >> .env

# Set your domain
echo "AUTHENTIK_ERROR_REPORTING__ENABLED=false" >> .env

cat .env
```

Edit `docker-compose.yml` to set the email configuration (used for password reset):

```yaml
# Add under the server service environment section
environment:
  AUTHENTIK_EMAIL__HOST: smtp.example.com
  AUTHENTIK_EMAIL__PORT: 587
  AUTHENTIK_EMAIL__USERNAME: noreply@example.com
  AUTHENTIK_EMAIL__PASSWORD: your-smtp-password
  AUTHENTIK_EMAIL__USE_TLS: "true"
  AUTHENTIK_EMAIL__FROM: "Authentik <noreply@example.com>"
```

Start the stack:
```bash
docker compose pull
docker compose up -d
docker compose logs -f
```

Authentik starts on port 9000 (HTTP) and 9443 (HTTPS). Use a reverse proxy for public access.

## Step 2: Configure Reverse Proxy (Caddy)

```
# /etc/caddy/Caddyfile
auth.example.com {
    reverse_proxy localhost:9000
}
```

Or Nginx:
```nginx
server {
    listen 443 ssl;
    server_name auth.example.com;

    ssl_certificate /etc/letsencrypt/live/auth.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/auth.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Step 3: Initial Setup

1. Navigate to `https://auth.example.com/if/flow/initial-setup/`
2. Create your admin account with a strong password
3. Log in at `https://auth.example.com`

## Step 4: Create an OIDC Provider for a Service

OIDC (OpenID Connect) lets applications delegate authentication to Authentik.

In the Authentik admin panel:

1. Go to **Applications** > **Providers** > **Create**
2. Select **OAuth2/OpenID Provider**
3. Configure:
 - Name: `grafana-provider`
 - Authorization flow: `default-provider-authorization-explicit-consent`
 - Client type: `Confidential`
 - Redirect URIs: `https://grafana.example.com/login/generic_oauth`
 - Copy the **Client ID** and **Client Secret**

4. Go to **Applications** > **Create**:
 - Name: `Grafana`
 - Slug: `grafana`
 - Provider: `grafana-provider`

### Connect Grafana to Authentik OIDC

In `/etc/grafana/grafana.ini` (or Grafana's environment variables):

```ini
[auth.generic_oauth]
enabled = true
name = Authentik
allow_sign_up = true
client_id = YOUR_CLIENT_ID
client_secret = YOUR_CLIENT_SECRET
scopes = openid email profile
auth_url = https://auth.example.com/application/o/grafana/authorize/
token_url = https://auth.example.com/application/o/grafana/token/
api_url = https://auth.example.com/application/o/userinfo/
```

## Step 5: Proxy Outpost (Protect Any HTTP App)

The Authentik proxy outpost can add authentication to any HTTP service, even ones that have no native SSO support.

In Authentik admin:
1. Go to **Applications** > **Providers** > **Create**
2. Select **Proxy Provider**
3. Configure:
 - Name: `wiki-proxy`
 - Mode: `Forward Auth (Single application)`
 - External host: `https://wiki.example.com`
 - Internal host: `http://localhost:3001`
4. Go to **Applications** > **Outposts** > **Create**:
 - Type: `Proxy`
 - Add `wiki-proxy` to Integration Applications
 - Deploy the outpost container:

```bash
# The outpost token is shown in the Outpost configuration
docker run -d \
  -e AUTHENTIK_HOST=https://auth.example.com \
  -e AUTHENTIK_INSECURE=false \
  -e AUTHENTIK_TOKEN=your-outpost-token \
  -p 9001:9001 \
  ghcr.io/goauthentik/proxy:latest
```

Configure Nginx to forward auth checks to the outpost:

```nginx
server {
    server_name wiki.example.com;

    location /outpost.goauthentik.io {
        proxy_pass http://localhost:9001;
        proxy_set_header Host $host;
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        add_header Set-Cookie $auth_cookie;
    }

    location / {
        auth_request /outpost.goauthentik.io/akprox/auth/nginx;
        error_page 401 = @authentik;
        auth_request_set $auth_cookie $upstream_http_set_cookie;
        add_header Set-Cookie $auth_cookie;

        proxy_pass http://localhost:3001;
    }

    location @authentik {
        return 302 /outpost.goauthentik.io/akprox/start;
    }
}
```

## Step 6: MFA Enrollment

1. In Authentik admin, go to **Flows & Stages** > **Stages** > **Create**
2. Add a **TOTP Setup Stage**
3. Add it to your `default-authentication-flow`

Users will be prompted to enroll a TOTP app (Google Authenticator, Aegis, etc.) on next login.

To force MFA for all users:
1. Go to **Directory** > **Groups** > Select "authentik Admins" or all users
2. Set the MFA policy on the group

## Step 7: Monitoring Authentik Events

Authentik logs all authentication events. View them at:
- **Events** > **Event Log** in the admin panel

Export to an external SIEM:
```bash
# Authentik exposes events via its REST API
curl -H "Authorization: Bearer YOUR_API_TOKEN" \
  "https://auth.example.com/api/v3/events/events/?action=login_failed&page_size=50" | \
  jq '.results[] | {user: .user.username, ip: .client_ip, timestamp: .created}'
```

## Step 8: LDAP Outpost for Legacy Applications

Some applications (Nextcloud, Gitea, older enterprise apps) do not support OIDC or SAML — they only speak LDAP. Authentik's LDAP outpost acts as an LDAP server backed by Authentik's user store, so legacy apps can authenticate against it without a separate LDAP server.

```bash
# Deploy the LDAP outpost
docker run -d \
  -e AUTHENTIK_HOST=https://auth.example.com \
  -e AUTHENTIK_INSECURE=false \
  -e AUTHENTIK_TOKEN=your-outpost-token \
  -p 389:3389 \   # map standard LDAP port to outpost's port
  -p 636:6636 \   # LDAPS
  ghcr.io/goauthentik/ldap:latest
```

In Authentik admin, create an LDAP provider:
1. **Applications → Providers → Create → LDAP Provider**
2. Base DN: `dc=ldap,dc=example,dc=com`
3. Bind Flow: `default-authentication-flow`
4. Search group: pick an Authentik group whose members can read the LDAP tree

Configure Nextcloud to use Authentik LDAP:
```
LDAP Host: your-server-ip
LDAP Port: 389
Base DN: dc=ldap,dc=example,dc=com
User DN: cn=ldapservice,ou=users,dc=ldap,dc=example,dc=com
Bind password: the-ldap-bind-password-from-authentik
User filter: (objectClass=user)
Login attribute: cn
```

---

## Step 9: Backup and Restore Authentik

Authentik stores everything in PostgreSQL. Back it up with `pg_dump`:

```bash
#!/bin/bash
# /usr/local/bin/backup-authentik.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/authentik"
mkdir -p "$BACKUP_DIR"

# Dump PostgreSQL from the authentik-postgresql container
docker exec authentik-postgresql-1 pg_dump -U authentik authentik \
  | gzip > "${BACKUP_DIR}/authentik-db-${DATE}.sql.gz"

# Also backup the media directory (profile pictures, certificates)
tar czf "${BACKUP_DIR}/authentik-media-${DATE}.tar.gz" \
  /opt/authentik/media/ 2>/dev/null || true

# Retain last 30 backups
find "$BACKUP_DIR" -name "*.sql.gz" -o -name "*.tar.gz" | sort | head -n -30 | xargs rm -f

echo "Authentik backup complete: ${BACKUP_DIR}/authentik-db-${DATE}.sql.gz"
```

Restore:
```bash
# Stop Authentik, restore the DB, restart
docker compose -f /opt/authentik/docker-compose.yml stop server worker

gunzip -c /backups/authentik/authentik-db-20260322_120000.sql.gz | \
  docker exec -i authentik-postgresql-1 psql -U authentik authentik

docker compose -f /opt/authentik/docker-compose.yml start server worker
```

---

## Related Reading

- [How to Set Up Caddy with Automatic HTTPS](/privacy-tools-guide/how-to-set-up-caddy-with-automatic-https/)
- [How to Set Up Nebula Mesh VPN for Teams](/privacy-tools-guide/how-to-set-up-nebula-mesh-vpn-for-teams/)
- [Secure Code Signing Setup for Developers](/privacy-tools-guide/secure-code-signing-setup-for-developers/)

- [How To Set Up Mobile Device Management Profile For Personal](/privacy-tools-guide/how-to-set-up-mobile-device-management-profile-for-personal-/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [AI Tools for Automated Secrets Rotation and Vault Management](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-automated-secrets-rotation-and-vault-management/)
---

## Related Articles

- [Set Up Mail In A Box Private Email Server Complete 2026](/privacy-tools-guide/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Best Privacy-Focused Email Aliases Service Comparison 2026](/privacy-tools-guide/best-privacy-focused-email-aliases-service-comparison-2026/)
- [How to Check If Your Email Server Has Been Blacklisted](/privacy-tools-guide/how-to-check-if-your-email-server-has-been-blacklisted/)
- [1Password Masked Email Feature Review: A Developer Guide](/privacy-tools-guide/1password-masked-email-feature-review/)
- [How to Set Up Self-Hosted Password Manager Vaultwarden 2026](/privacy-tools-guide/how-to-set-up-self-hosted-password-manager-vaultwarden-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
