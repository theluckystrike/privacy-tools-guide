---
layout: default
title: "Nextcloud Collabora Office Setup Guide"
description: "A complete technical guide for integrating Collabora Online with Nextcloud. Covers Docker deployment, reverse proxy configuration, SSL setup, and."
date: 2026-03-15
author: theluckystrike
permalink: /nextcloud-collabora-office-setup-guide/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
---

{% raw %}

Deploy Collabora Online with your Nextcloud instance to enable real-time document editing, spreadsheets, and presentations directly in your private cloud. This guide covers the complete setup process using Docker, nginx reverse proxy configuration, and security hardening for production deployments.

## Why Collabora Office with Nextcloud

Collabora Online provides a self-hosted alternative to Google Docs or Microsoft 365. When integrated with Nextcloud, you gain full control over your documents without sending data to third-party services. The integration supports real-time collaboration, version history, and the full LibreOffice feature set.

For developers, this setup demonstrates reverse proxy configuration, Docker networking, and secure internal service communication. For organizations, it delivers enterprise document editing while maintaining data sovereignty.

## Prerequisites

Before starting, verify these requirements:

- Nextcloud instance running (version 24 or newer recommended)
- A domain name with DNS configured
- Docker and Docker Compose installed on your server
- nginx as your reverse proxy (other web servers work but this guide uses nginx)
- SSL certificates (Let's Encrypt recommended)

## Step 1: Deploy Collabora Online via Docker

Create a Docker Compose file for Collabora. The official Collabora image includes everything needed for the office suite.

```bash
mkdir -p /opt/collabora && cd /opt/collabora
```

Create the `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  collabora:
    image: collabora/code:24.04.5.2.1
    container_name: collabora-online
    restart: unless-stopped
    ports:
      - "9980:9980"
    environment:
      - domain=cloud\\.yourdomain\\.com
      - username=admin
      - password=your-secure-password
      - DONT_GEN_SSL_CERT=1
      - extra_params=--o:security.enableTLS=false
    cap_add:
      - MKNOD
    volumes:
      - ./loolwsd.xml:/etc/loolwsd/loolwsd.xml
    networks:
      - office-network

networks:
  office-network:
    name: nextcloud-network
    external: true
```

Replace `cloud\\.yourdomain\\.com` with your Nextcloud domain, escaping dots for the regex pattern. Generate a strong password for the admin user.

Create the shared network if it doesn't exist:

```bash
docker network create nextcloud-network 2>/dev/null || true
```

Start the container:

```bash
docker-compose up -d
```

Verify the container is running:

```bash
docker ps | grep collabora
```

## Step 2: Configure nginx Reverse Proxy

Set up nginx to forward HTTPS requests to the Collabora container. This separates the office backend from your main Nextcloud URL while maintaining SSL termination.

Create the nginx configuration:

```nginx
server {
    listen 443 ssl http2;
    server_name office.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/office.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/office.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # WebSocket support for real-time collaboration
        proxy_http_version 1.1;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # LOOL WSD specific headers
    location /lool {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_http_version 1.1;
    }
}
```

Enable the site and test the configuration:

```bash
ln -s /etc/nginx/sites-available/collabora /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## Step 3: Install and Configure the Nextcloud App

Install the Collabora Online app through the Nextcloud web interface or via occ:

```bash
cd /var/www/nextcloud
occ app:install richdocuments
```

After installation, configure the server URL in Nextcloud settings:

1. Navigate to **Settings** → **Administration** → **Office**
2. Enter your Collabora URL: `https://office.yourdomain.com`
3. Enable "Use your own server"
4. Save the configuration

Alternatively, set this via the command line:

```bash
occ config:app:set richdocuments wopi_url --value="https://office.yourdomain.com"
occ config:app:set richdocuments disable_certificate_verification --value="yes"
```

## Step 4: Security Hardening

For production deployments, apply these security measures:

### Restrict Access by IP

Edit the Collabora container environment in `docker-compose.yml`:

```yaml
environment:
  - domain=cloud\\.yourdomain\\.com
  - allowed_origin=cloud.yourdomain.com
```

### Enable SSL in Production

Remove the `DONT_GEN_SSL_CERT` setting and ensure your reverse proxy handles SSL properly:

```yaml
environment:
  - domain=cloud\\.yourdomain\\.com
  - username=admin
  - password=your-secure-password
```

### Configure WOPI Isolation

Edit `/etc/loolwsd/loolwsd.xml` inside the container or mount a custom configuration:

```xml
<net>
    <alias_group>
        <host desc="Allow nextcloud.yourdomain.com">nextcloud.yourdomain.com</host>
        <host desc="Allow office.yourdomain.com">office.yourdomain.com</host>
    </alias_group>
</net>
<security>
    <capabilities>
        <!-- Restrict file capabilities -->
    </capabilities>
</security>
```

Restart the container after changes:

```bash
docker-compose down && docker-compose up -d
```

## Step 5: Verify the Integration

Test your setup by:

1. Logging into Nextcloud
2. Creating a new document (Text, Spreadsheet, or Presentation)
3. Opening the document and verifying it loads the Collabora editor
4. Making edits and saving to confirm file operations work

Check container logs if issues occur:

```bash
docker logs collabora-online
```

Monitor nginx access logs for connection issues:

```tail -f /var/log/nginx/office.access.log```

## Troubleshooting Common Issues

### Document Fails to Load

Check that the domain in your Collabora configuration matches your Nextcloud URL exactly. Dots must be escaped in the `domain` environment variable.

### WebSocket Connection Errors

Verify nginx has WebSocket proxy headers configured correctly. The `Upgrade` and `Connection` headers are required for real-time collaboration.

### Slow Document Loading

Increase container memory limits:

```yaml
deploy:
 resources:
 limits:
 memory: 4G
```

### SSL Certificate Errors

Ensure your reverse proxy SSL certificates are valid. For testing, you can disable certificate verification in Nextcloud:

```bash
occ config:app:set richdocuments disable_certificate_verification --value="yes"
```

## Performance Optimization

For deployments with multiple users, consider these optimizations:

- Increase container memory allocation to 4GB or more
- Use a separate PostgreSQL database for Collabora (enterprise feature)
- Implement caching with Redis for session management
- Configure nginx caching for static assets

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
 proxy_pass http://127.0.0.1:9980;
 proxy_cache_valid 200 1h;
 add_header Cache-Control "public, immutable";
}
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
