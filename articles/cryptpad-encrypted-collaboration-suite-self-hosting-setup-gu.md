---
layout: default
title: "Cryptpad Encrypted Collaboration Suite Self Hosting Setup Gu"
description: "Learn how to self-host CryptPad for end-to-end encrypted document collaboration. This practical guide covers Docker deployment, configuration, security"
date: 2026-03-16
author: theluckystrike
permalink: /cryptpad-encrypted-collaboration-suite-self-hosting-setup-gu/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, collaboration]
---

{% raw %}

CryptPad is an end-to-end encrypted collaboration suite that provides real-time document editing, spreadsheets, presentations, and more without sacrificing privacy. Unlike conventional office suites, CryptPad encrypts all data on the client side—your server never sees plaintext content. Self-hosting gives teams full control over their data while maintaining CryptPad's zero-knowledge architecture.

This guide walks through deploying CryptPad using Docker, configuring essential settings, hardening security, and managing team access.

## Prerequisites

Before starting, ensure you have:

- A Linux server with at least 2GB RAM and 20GB storage
- Docker and Docker Compose installed
- A domain pointing to your server (for TLS certificates)
- Basic familiarity with the command line

## Installing CryptPad with Docker

The recommended approach uses Docker for isolated deployment and simplified updates. Create a directory for your installation:

```bash
mkdir -p ~/cryptpad && cd ~/cryptpad
```

Create your Docker Compose configuration:

```yaml
version: '3.8'

services:
  cryptpad:
    image: cryptpad/cryptpad:latest
    container_name: cryptpad
    ports:
      - "3000:3000"
    volumes:
      - ./customize:/var/cryptpad/customize
      - ./data:/var/cryptpad/data
    environment:
      - CPAD_MAIN_DOMAIN=cryptpad.yourdomain.com
      - CPAD_SMTP_URL=smtp://user:pass@smtp.example.com:587
      - CPAD_SMTP_FROM=CryptPad <noreply@yourdomain.com>
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"]
      interval: 30s
      timeout: 10s
      retries: 3
```

Start the container:

```bash
docker-compose up -d
```

After startup, access CryptPad at `http://localhost:3000`. Configure a reverse proxy (nginx, Caddy, or Traefik) with TLS to handle HTTPS. Caddy simplifies this with automatic HTTPS:

```yaml
# Add to docker-compose.yml
caddy:
  image: caddy:2
  container_name: cryptpad-caddy
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile
    - ./data/caddy:/data
  depends_on:
    - cryptpad
```

Corresponding Caddyfile:

```
cryptpad.yourdomain.com {
    reverse_proxy cryptpad:3000
    encode gzip
}
```

## Understanding CryptPad's Encryption Model

CryptPad employs a unique encryption architecture worth understanding before production deployment. Each document generates a random 256-bit key locally. The URL itself contains the encryption key—sharing a document URL grants decryption access. This design means your server stores only encrypted blobs; even with full server access, administrators cannot read user content.

User accounts use a different mechanism. When users create accounts, CryptPad derives encryption keys from their password using Argon2id. The server stores authentication credentials separately from encryption keys. This separation ensures that compromising the database does not automatically expose document contents.

## Configuration and Customization

CryptPad stores configuration in `/var/cryptpad/customize` when using Docker. Key files include:

**Limit Registration:**

```bash
echo '{"limitRegistration": true}' > customize/config.json
```

This prevents public signup. Combine with admin invite codes for controlled deployments.

**Admin Mode:**

To access admin features, create an admin user account, then run:

```bash
docker exec cryptpad node bin/admin.js addadmin user@yourdomain.com
```

Admin users can create invite codes, manage storage quotas, and access usage statistics.

**Storage Quotas:**

Set per-user storage limits in `customize/config.json`:

```json
{
  "limitRegistration": true,
  "maxUploadSize": 52428800,
  "defaultStorage": 104857600
}
```

This limits uploads to 50MB and grants each user 100MB by default.

## Security Hardening

Production deployments require attention to several security areas:

**Disable Remote Logging:**

By default, CryptPad may send error reports. Disable this in production:

```bash
echo '{"sentry": false}' > customize/config.json
```

**Configure Content Security Policy:**

Add custom CSP headers through your reverse proxy. For nginx:

```
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
```

**Database Encryption:**

While CryptPad encrypts document content, user credentials and metadata remain. Consider enabling disk encryption at the filesystem level:

```bash
# For Ubuntu with LUKS
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 cryptpad_data
sudo mkfs.ext4 /dev/mapper/cryptpad_data
```

Mount the encrypted volume at `/var/lib/docker/volumes/cryptpad_data` before starting containers.

**Network Isolation:**

Run CryptPad in a dedicated network namespace:

```bash
docker network create --driver bridge cryptpad_net
# Add to docker-compose.yml networks section
networks:
  - cryptpad_net
```

## Team User Management

Self-hosted CryptPad offers several user management strategies:

**Invite Codes:**

Generate invite codes for controlled registration:

```bash
docker exec cryptpad node bin/admin.js newinvite --count 10 --expiry 30d
```

This creates 10 codes valid for 30 days. Distribute codes to team members.

**LDAP Integration:**

For larger organizations, CryptPad supports LDAP authentication. Enable in `customize/config.json`:

```json
{
  "ldap": {
    "url": "ldap://ldap.yourdomain.com",
    "bindDn": "cn=admin,dc=example,dc=com",
    "bindCredentials": "your-password",
    "searchBase": "ou=users,dc=example,dc=com",
    "searchFilter": "(uid={{username}})"
  }
}
```

Replace `your-password` with an environment variable or secrets management solution.

**Storage Management:**

Monitor storage with admin scripts:

```bash
# List users and storage usage
docker exec cryptpad node bin/admin.js listusers

# Set specific user quota
docker exec cryptpad node bin/admin.js setquota user@domain.com 209715200
```

## Backup and Recovery

Implement regular backups of the CryptPad data volume:

```bash
#!/bin/bash
# backup-cryptpad.sh
DATE=$(date +%Y%m%d)
docker run --rm -v cryptpad_data:/data -v $(pwd):/backup alpine tar czf /backup/cryptpad-backup-$DATE.tar.gz /data
```

Schedule daily backups with cron:

```bash
0 2 * * * /path/to/backup-cryptpad.sh
```

Test restoration periodically:

```bash
# Stop container
docker-compose down
# Restore
docker run --rm -v cryptpad_data:/data -v $(pwd):/backup alpine tar xzf /backup/cryptpad-backup-20260316.tar.gz -C /
# Start container
docker-compose up -d
```

## Performance Considerations

For teams exceeding 50 users, adjust these settings in `customize/config.json`:

```json
{
  "worker": true,
  "httpUnsafeOrigin": "https://cryptpad.yourdomain.com",
  "adminEmail": "admin@yourdomain.com"
}
```

Consider enabling Redis for session management in high-load scenarios by adding a Redis service to your compose file and configuring `customize/redis.json`.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
