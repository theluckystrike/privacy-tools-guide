---
layout: default
title: "How to Set Up Authelia for SSO"
description: "Deploy Authelia as a self-hosted SSO and two-factor authentication gateway for your internal services using Docker, Traefik, and LDAP or a flat user file"
date: 2026-03-22
author: theluckystrike
permalink: /authelia-sso-setup-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
version: '3.8'
services: authelia:
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
networks: proxy:
    external: true
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

## Migrating from File-Based to LDAP Authentication

The flat `users_database.yml` works for small self-hosted setups. When you need centralized user management across multiple services, switch to LDAP. This lets you add and remove users in one place rather than editing YAML files.

Replace the `authentication_backend` block in `configuration.yml`:

```yaml
authentication_backend:
  ldap:
    url: ldap://lldap:3890
    base_dn: DC=yourdomain,DC=com
    username_attribute: uid
    additional_users_dn: OU=people
    users_filter: "(&({username_attribute}={input})(objectClass=person))"
    additional_groups_dn: OU=groups
    groups_filter: "(member={dn})"
    group_name_attribute: cn
    mail_attribute: mail
    display_name_attribute: displayName
    user: CN=readonly,OU=people,DC=yourdomain,DC=com
    password: "readonly-bind-password"
```

`lldap` (Lightweight LDAP) is a popular Docker container for self-hosted LDAP:

```yaml
# Add to docker-compose.yml
lldap:
  image: lldap/lldap:stable
  container_name: lldap
  restart: unless-stopped
  environment:
    - LLDAP_JWT_SECRET=your-jwt-secret
    - LLDAP_LDAP_USER_PASS=admin-password
    - LLDAP_LDAP_BASE_DN=dc=yourdomain,dc=com
  volumes:
    - ./lldap-data:/data
  networks:
    - proxy
```

Once lldap is running, use its web UI at port 17170 to manage users and groups. Authelia reads the LDAP directory without a local file to manage.

## Authelia with OAuth2 / OpenID Connect

Authelia 4.35+ supports acting as an OpenID Connect provider. This means applications that support OIDC (Nextcloud, Gitea, Grafana) can delegate login to Authelia rather than requiring `forwardAuth`.

Add to `configuration.yml`:

```yaml
identity_providers:
  oidc:
    hmac_secret: "your-hmac-secret-32-chars-minimum"
    issuer_private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      (generate with: openssl genrsa 4096)
      -----END RSA PRIVATE KEY-----

    clients:
      - id: nextcloud
        description: Nextcloud
        secret: "client-secret-here"
        public: false
        authorization_policy: two_factor
        redirect_uris:
          - https://nextcloud.yourdomain.com/apps/user_oidc/code
        scopes:
          - openid
          - profile
          - email
          - groups
        userinfo_signing_algorithm: none

      - id: grafana
        description: Grafana
        secret: "another-client-secret"
        public: false
        authorization_policy: one_factor
        redirect_uris:
          - https://grafana.yourdomain.com/login/generic_oauth
        scopes:
          - openid
          - profile
          - email
```

In Grafana's `grafana.ini`, configure:

```ini
[auth.generic_oauth]
enabled = true
name = Authelia
client_id = grafana
client_secret = another-client-secret
scopes = openid profile email groups
auth_url = https://auth.yourdomain.com/api/oidc/authorization
token_url = https://auth.yourdomain.com/api/oidc/token
api_url = https://auth.yourdomain.com/api/oidc/userinfo
```

OIDC is more secure than forwardAuth for apps that support it — the application handles the token exchange directly rather than relying on proxy headers.

## Related Articles

- [How to Set Up Authentik for Identity Management](/how-to-set-up-authentik-for-identity-management/)
- [Self-Hosted Private Git Server with Gitea](/private-git-server-gitea-setup-guide/)
- [How To Set Up Dnscrypt Proxy For Authenticated Encrypted](/how-to-set-up-dnscrypt-proxy-for-authenticated-encrypted-dns/)
- [Nextcloud Collabora Office Setup Guide](/nextcloud-collabora-office-setup-guide/)
- [Jitsi Meet Self Hosted Setup Guide](/jitsi-meet-self-hosted-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
