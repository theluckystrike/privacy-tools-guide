---
layout: default
title: "How to Self-Host Bitwarden Vaultwarden: Complete Setup Guide"
description: "Complete Vaultwarden self-hosting guide. Docker setup, reverse proxy, SSL, backups, updates, and security hardening for password vault."
date: 2026-03-20
author: theluckystrike
permalink: /how-to-self-host-bitwarden-vaultwarden-complete-setup-guide/
categories: [guides]
tags: [privacy-tools, self-hosted, bitwarden, vaultwarden, security]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

Vaultwarden is a lightweight, open-source implementation of Bitwarden's password vault API. Self-hosting Vaultwarden gives you full control over your credentials without trusting a third-party SaaS provider. This guide covers full Docker deployment, reverse proxy configuration with Nginx, SSL certificate automation with Let's Encrypt, automated backups, security hardening, and the maintenance workflow for running your own password vault.

## Why Self-Host Vaultwarden?

Bitwarden's SaaS offering is privacy-respecting, but self-hosting provides:
- **Zero trust dependency:** Your encryption keys never leave your server
- **Full data ownership:** No vendor lock-in, easy migration
- **Cost savings:** Free (Docker) vs. $2.99/month Bitwarden Premium
- **Compliance:** Required for regulated industries (healthcare, finance)
- **Custom features:** Add integrations, modify workflows

Trade-off: You manage infrastructure, backups, SSL certificates, and server updates.

## System Requirements

**Minimum specs for single-user Vaultwarden:**
- 512 MB RAM
- 5 GB disk space (includes database + backups)
- 1 vCPU (or ARM equivalent, Raspberry Pi 4 works)
- Ubuntu 20.04 LTS or later, Debian 11+, or macOS with Docker Desktop

**For multiple users (family, small team):**
- 2 GB RAM
- 20 GB disk space
- 2 vCPU
- 25-50 Mbps connection (upload for backups)

**Deployment options:**
- VPS (DigitalOcean $6-12/month, Linode $5+/month, Vultr $2.50+/month)
- Home server (Raspberry Pi 4, old laptop, NAS)
- Synology NAS (Docker support built-in)

## Installation: Docker Compose Setup

**Step 1: Install Docker**

On Ubuntu/Debian:
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add current user to docker group (avoid sudo for docker commands)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker run hello-world
```

On macOS:
```bash
# Install via Homebrew
brew install docker-compose

# Or download Docker Desktop from docker.com
# Docker Desktop includes docker and docker-compose
```

**Step 2: Create directory structure**

```bash
# Create vaultwarden directory
mkdir -p ~/vaultwarden/{data,nginx,letsencrypt}
cd ~/vaultwarden

# Create subdirectories
mkdir -p data/attachments
mkdir -p nginx/ssl
```

**Step 3: Create Docker Compose configuration**

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "false"
      SHOW_PASSWORD_HINT: "false"
      LOG_LEVEL: "info"
      LOG_FILE: "/data/vaultwarden.log"
      EXTENDED_LOGGING: "true"
      EXTENDED_LOGGING_FORMAT: "json"

      # Admin panel token (change this to a long random string)
      ADMIN_TOKEN: "$argon2id$v=19$m=19456,t=2,p=1$your-long-random-string-here"

      # Database (SQLite for single-user, PostgreSQL for scale)
      DATABASE_URL: "sqlite:///data/db.sqlite3"

      # Email notifications (optional)
      # SMTP_HOST: "smtp.gmail.com"
      # SMTP_PORT: "587"
      # SMTP_SECURITY: "starttls"
      # SMTP_USERNAME: "your-email@gmail.com"
      # SMTP_PASSWORD: "your-app-password"

    ports:
      - "127.0.0.1:8080:80"

    volumes:
      - ./data:/data
      - ./data/attachments:/data/attachments
      - ./data/vaultwarden.log:/data/vaultwarden.log

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/alive"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 30s

  # PostgreSQL (optional, for scale and reliability)
  # postgres:
  #   image: postgres:15-alpine
  #   container_name: vaultwarden-db
  #   restart: always
  #   environment:
  #     POSTGRES_USER: vaultwarden
  #     POSTGRES_PASSWORD: "long-random-password"
  #     POSTGRES_DB: vaultwarden
  #   volumes:
  #     - ./data/postgres:/var/lib/postgresql/data
  #   ports:
  #     - "127.0.0.1:5432:5432"

networks:
  default:
    name: vaultwarden-network
```

**Generate admin token:**

```bash
# Generate a strong admin token
openssl rand -base64 32

# Use with htpasswd to create Argon2id hash
sudo apt install apache2-utils  # or brew install httpd
htpasswd -c -B -C 10 /tmp/admin admin
# Copy the hash to ADMIN_TOKEN in docker-compose.yml
```

Or use an online generator (less secure): [Argon2 generator](https://argon2.online)

**Step 4: Start Vaultwarden**

```bash
# Navigate to vaultwarden directory
cd ~/vaultwarden

# Start the container
docker-compose up -d

# View logs
docker-compose logs -f vaultwarden

# Expected output:
# [INFO] vaultwarden 1.0.37
# [INFO] Rocket has launched from http://0.0.0.0
# [INFO] Database: sqlite:///data/db.sqlite3
```

Vaultwarden is now running on `http://localhost:8080` (local access only).

## Reverse Proxy: Nginx Configuration

Running Vaultwarden directly on the internet is risky. Use Nginx as a reverse proxy to:
- Add SSL/TLS encryption
- Rate-limit API requests
- Add security headers
- Separate concerns (web server vs. application)

**Step 1: Create Nginx configuration**

Create `nginx/vaultwarden.conf`:

```nginx
# Upstream vaultwarden server
upstream vaultwarden {
    server vaultwarden:80;
    keepalive 32;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=vaultwarden_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=identity_limit:10m rate=5r/m;

server {
    listen 80;
    server_name vault.example.com www.vault.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name vault.example.com www.vault.example.com;

    # SSL certificates (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/vault.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vault.example.com/privkey.pem;

    # Modern SSL configuration
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # Logging
    access_log /var/log/nginx/vaultwarden_access.log;
    error_log /var/log/nginx/vaultwarden_error.log;

    # Body size limit (for file uploads)
    client_max_body_size 525M;

    # Root location
    root /var/www/html;
    index index.html;

    # Main reverse proxy
    location / {
        proxy_pass http://vaultwarden;
        proxy_http_version 1.1;

        # Headers for WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;

        # Timeouts for long-running operations
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # API rate limiting
    location /identity {
        limit_req zone=identity_limit burst=2 nodelay;
        proxy_pass http://vaultwarden;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api {
        limit_req zone=vaultwarden_limit burst=20 nodelay;
        proxy_pass http://vaultwarden;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Block admin panel from public internet
    location /admin {
        allow 127.0.0.1;
        allow 192.168.1.0/24;  # Your home network
        deny all;

        proxy_pass http://vaultwarden;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Step 2: Add Nginx to Docker Compose**

Update `docker-compose.yml`:

```yaml
  nginx:
    image: nginx:latest
    container_name: vaultwarden-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/vaultwarden.conf:/etc/nginx/conf.d/vaultwarden.conf:ro
      - ./letsencrypt:/etc/letsencrypt:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - vaultwarden
    networks:
      - vaultwarden-network
```

## SSL Certificates: Let's Encrypt Setup

**Option 1: Certbot (recommended for dynamic DNS)**

Install Certbot on your host machine (not in Docker):

```bash
# Ubuntu/Debian
sudo apt install certbot python3-certbot-nginx

# macOS
brew install certbot

# Request initial certificate
sudo certbot certonly --standalone -d vault.example.com -d www.vault.example.com

# Expected output:
# Saving debug log to /var/log/letsencrypt/letsencrypt.log
# Certificate is saved at: /etc/letsencrypt/live/vault.example.com/fullchain.pem
# Key is saved at: /etc/letsencrypt/live/vault.example.com/privkey.pem
```

**Auto-renew with cron job:**

```bash
# Edit crontab
sudo crontab -e

# Add line (runs daily at 2 AM):
0 2 * * * certbot renew --quiet && systemctl reload nginx
```

**Option 2: Docker-based (Certbot in container)**

Add to docker-compose.yml:

```yaml
  certbot:
    image: certbot/certbot:latest
    container_name: vaultwarden-certbot
    restart: always
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./certbot/renewal:/var/lib/letsencrypt
    entrypoint: /bin/sh -c "certbot renew --quiet && sleep 86400 && exec /bin/sh -c \"$$@\"" -- /bin/sh
    environment:
      CERTBOT_EMAIL: "admin@example.com"
```

## Backups: Automated Daily Backups

Create backup script `backup.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="./backups"
DATA_DIR="./data"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/vaultwarden_backup_$TIMESTAMP.tar.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database, config, and attachments
tar -czf $BACKUP_FILE \
    -C $DATA_DIR db.sqlite3 config.json 2>/dev/null || \
    tar -czf $BACKUP_FILE \
    -C $DATA_DIR . \
    --exclude='*.log'

# Log backup
echo "[$(date)] Backup created: $BACKUP_FILE ($(du -h $BACKUP_FILE | cut -f1))" >> $BACKUP_DIR/backup.log

# Remove old backups (keep last 30 days)
find $BACKUP_DIR -name "vaultwarden_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Optional: Upload to cloud storage
# aws s3 cp $BACKUP_FILE s3://your-backup-bucket/vaultwarden/

echo "Backup complete."
```

**Add daily backup cron job:**

```bash
# Make script executable
chmod +x backup.sh

# Edit crontab
crontab -e

# Add line (runs daily at 3 AM):
0 3 * * * /home/user/vaultwarden/backup.sh >> /tmp/vaultwarden_backup.log 2>&1
```

**Test backup recovery:**

```bash
# Extract backup
tar -xzf ./backups/vaultwarden_backup_20260320_030000.tar.gz

# Verify files are intact
ls -la ./data/
```

## Security Hardening

**1. Disable signups (if self-hosted for one user):**

Already set in docker-compose.yml: `SIGNUPS_ALLOWED: "false"`

**2. Change admin token:**

Generate a new token and restart:

```bash
# Generate token
openssl rand -base64 32

# Update docker-compose.yml ADMIN_TOKEN
# Restart service
docker-compose restart vaultwarden
```

**3. Firewall rules:**

```bash
# UFW (Ubuntu)
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# iptables (direct)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -j DROP
```

**4. Fail2ban (rate-limit login attempts):**

```bash
# Install
sudo apt install fail2ban

# Create filter /etc/fail2ban/filter.d/vaultwarden.conf:
[Definition]
failregex = ^.* Invalid username or password\. \[.*?\] \[<HOST>\] .* -$
ignoreregex =

# Create jail /etc/fail2ban/jail.d/vaultwarden.conf:
[vaultwarden]
enabled = true
port = http,https
filter = vaultwarden
logpath = /path/to/vaultwarden.log
maxretry = 5
findtime = 3600
bantime = 604800

# Restart fail2ban
sudo systemctl restart fail2ban
```

**5. Monitor logs:**

```bash
# Follow logs in real-time
docker-compose logs -f vaultwarden

# Check for suspicious activity
docker-compose logs vaultwarden | grep -i "error\|invalid\|unauthorized"
```

## Maintenance and Updates

**Updating Vaultwarden:**

```bash
# Pull latest image
docker pull vaultwarden/server:latest

# Restart container (will use new image)
docker-compose up -d vaultwarden

# View version
docker-compose logs vaultwarden | grep "vaultwarden"
```

**Monitoring disk space:**

```bash
# Check backup size
du -sh ./backups/

# Check database size
du -sh ./data/db.sqlite3

# Alert if backup grows too large
find ./backups -name "*.tar.gz" -mtime -1 -exec du -sh {} \; | awk '{print $1}'
```

**Database maintenance:**

```bash
# Optimize SQLite database (monthly)
# Connect to container and run:
docker-compose exec vaultwarden sqlite3 /data/db.sqlite3 "VACUUM;"
```

## Client Setup

**Bitwarden Web Vault:** https://vault.example.com

**Mobile/Desktop Apps:**
1. Download Bitwarden app (iOS, Android, macOS, Windows, Linux)
2. In settings, set custom server: https://vault.example.com
3. Create account (or import from cloud Bitwarden)
4. Sync vault data

**Browser Extension:**
1. Install Bitwarden extension
2. Open extension settings
3. Set server URL: https://vault.example.com
4. Login with account credentials

## Troubleshooting

**Login fails with "connection refused":**
```bash
# Check if vaultwarden is running
docker-compose ps

# Restart if needed
docker-compose restart vaultwarden
```

**HTTPS certificate errors:**
```bash
# Check certificate expiration
openssl x509 -in /etc/letsencrypt/live/vault.example.com/fullchain.pem -text -noout | grep -E "Not Before|Not After"

# Manually renew
sudo certbot renew --force-renewal
```

**Slow performance:**
```bash
# Check resource usage
docker stats vaultwarden

# Increase memory limit in docker-compose.yml
# Restart container
docker-compose restart vaultwarden
```

**Database is locked:**
```bash
# Stop container
docker-compose stop vaultwarden

# Delete lock file
rm ./data/db.sqlite3-wal ./data/db.sqlite3-shm 2>/dev/null

# Restart
docker-compose start vaultwarden
```

## Cost and Alternatives

**Self-hosted Vaultwarden:**
- Cost: $0-5/month (VPS) + your time
- Pros: Full control, no recurring fees
- Cons: Maintenance responsibility

**Bitwarden Cloud:**
- Cost: $0 (free tier) or $2.99/month (premium)
- Pros: Managed, automatic updates
- Cons: Trust third-party

**Alternative password managers:**
- 1Password ($14.99/month): Excellent, closed-source
- LastPass ($36/year): Basic features, security history
- KeePass (free): Local-only, requires manual sync
- Pass (free): CLI-based, Unix philosophy

For privacy-conscious users who want control, self-hosted Vaultwarden is the best value.

## Related Reading

- [Privacy-First Password Manager Comparison](/password-manager-comparison/)
- [Self-Hosting Guide for Privacy-Conscious Users](/self-hosting-guide/)
- [SSL/TLS Certificates and HTTPS Setup](/https-setup-guide/)
- [Docker Security Best Practices](/docker-security-guide/)
- [Automated Backup Strategies for Self-Hosted Services](/backup-strategies-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
