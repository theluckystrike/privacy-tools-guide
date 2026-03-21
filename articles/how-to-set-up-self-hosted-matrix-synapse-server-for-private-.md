---
layout: default
title: "How To Set Up Self Hosted Matrix Synapse Server For Private"
description: "Matrix is an open protocol for real-time communication, and Synapse is the reference implementation of a Matrix homeserver. Running your own Synapse instance"
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-self-hosted-matrix-synapse-server-for-private-/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---


{% raw %}

Matrix is an open protocol for real-time communication, and Synapse is the reference implementation of a Matrix homeserver. Running your own Synapse instance gives you complete control over your messaging infrastructure, end-to-end encryption keys, and data retention policies. This guide walks through deploying a production-ready Synapse server using Docker, configuring essential security settings, and connecting your first clients.

## Prerequisites

Before starting, ensure you have a Linux server with at least 2GB RAM and a domain pointing to your server's IP address. You'll also need Docker and Docker Compose installed:

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
```

This guide assumes you have basic familiarity with the command line and a registered domain. Let's proceed with the installation.

## Installing Synapse with Docker

The recommended way to run Synapse is via Docker, which isolates the application and simplifies updates. Create a directory for your Synapse deployment:

```bash
mkdir -p ~/synapse && cd ~/synapse
```

Create a `docker-compose.yml` file with the following configuration:

```yaml
version: '3'

services:
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: synapse
    restart: unless-stopped
    ports:
      - "8008:8008"
      - "8448:8448"
    volumes:
      - ./data:/data
    environment:
      - SYNAPSE_SERVER_NAME=yourdomain.com
      - SYNAPSE_REPORT_STATS=no
```

Replace `yourdomain.com` with your actual domain. Generate the Synapse configuration:

```bash
cd ~/synapse
docker-compose run --rm synapse generate
```

This creates a `homeserver.yaml` file in the data directory. Review the generated configuration before starting the container:

```bash
docker-compose up -d
```

Verify the server is running:

```bash
docker logs synapse
```

You should see messages indicating Synapse has started successfully and is listening on ports 8008 (client API) and 8448 (federation).

## Configuring SSL/TLS

Matrix requires TLS for secure communications. For a production deployment, use a reverse proxy like Caddy or Nginx with automatic SSL certificates. Here's a Caddyfile configuration:

```
matrix.yourdomain.com {
    reverse_proxy localhost:8008
}
```

Caddy automatically obtains Let's Encrypt certificates. Alternatively, use Nginx with Certbot:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d matrix.yourdomain.com
```

Nginx configuration for Matrix:

```nginx
server {
    listen 443 ssl http2;
    server_name matrix.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/matrix.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.yourdomain.com/privkey.pem;

    location /_matrix {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
    }
}
```

## Creating Admin Users and Registering Clients

Create an admin user to manage your server:

```bash
docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

Follow the prompts to set username and password. This user has administrator privileges.

For client connections, you can use Element (formerly Riot.im), the reference Matrix client. Download Element at element.io and configure it to connect to your server:

1. Open Element and select "Edit"
2. Choose "Advanced" and enter your homeserver URL: `https://matrix.yourdomain.com`
3. Log in with the credentials you created above

## Enabling End-to-End Encryption

Matrix supports end-to-end encryption (E2EE) by default using the Olm and Megolm protocols. When you create a new room in Element, encryption is available by toggling the encryption setting in room settings. Each device gets its own encryption keys stored locally.

For additional security, verify key fingerprints in the settings under "Security & Privacy". This ensures you're communicating with the correct recipients and not victim to man-in-the-middle attacks.

## Hardening Your Synapse Server

Several configuration changes improve your server's security. Edit the `homeserver.yaml` file in your data directory:

### Disable Public Registration

Prevent unauthorized users from creating accounts:

```yaml
enable_registration: false
```

### Configure Cross-Origin Resource Sharing

Restrict API access:

```yaml
cors:
  enabled: false
```

### Set Up Automated Backups

Matrix stores data in SQLite (for small deployments) or PostgreSQL (recommended for production). For SQLite, create a backup script:

```bash
#!/bin/bash
cp /home/user/synapse/data/homeserver.db /backup/homeserver-$(date +%Y%m%d).db
```

Add this to cron for daily backups:

```bash
crontab -e
0 2 * * * /path/to/backup-script.sh
```

### Update Synapse Regularly

Stay current with security patches:

```bash
docker-compose pull
docker-compose up -d
```

## Testing Federation

One of Matrix's strengths is federation—servers communicating with each other. Test federation by creating a room and inviting a user from another Matrix server, such as matrix.org. If federation works, you'll see messages flow between servers.

To verify federation is working, check the server's federation API:

```bash
curl -X GET "https://matrix.yourdomain.com/_matrix/federation/v1/version"
```

A successful response indicates federation is operational.

## Troubleshooting Common Issues

If clients cannot connect, check your reverse proxy configuration and ensure ports 8008 and 8448 are open in your firewall:

```bash
sudo ufw allow 8008/tcp
sudo ufw allow 8448/tcp
sudo ufw reload
```

For slow performance, consider switching to PostgreSQL. Update your `docker-compose.yml`:

```yaml
services:
  synapse:
    image: matrixdotorg/synapse:latest
    environment:
      - SYNAPSE_DATABASE_TYPE=psycopg2
      - SYNAPSE_DATABASE_HOST=postgres
    depends_on:
      - postgres
    # ... rest of config

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=synapse
      - POSTGRES_PASSWORD=strongpassword
      - POSTGRES_DB=synapse
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
