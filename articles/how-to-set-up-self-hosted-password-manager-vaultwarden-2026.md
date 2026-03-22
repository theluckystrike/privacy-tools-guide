---
title: "How to Set Up Self-Hosted Password Manager Vaultwarden 2026"
date: 2026-03-21
author: "Privacy Tools Guide"
reviewed: true
score: 8
voice-checked: true
intent-checked: true
permalink: /how-to-set-up-self-hosted-password-manager-vaultwarden-2026/
description: "Follow this guide to how to set up self hosted password manager vaultwarden 2026 with practical examples, tips, and step-by-step instructions for getting"
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up Self-Hosted Password Manager Vaultwarden 2026

Vaultwarden is a free, open-source password manager compatible with Bitwarden clients. Unlike cloud-hosted password managers, you control the server—no company between you and your encrypted vault. This guide covers complete setup: Docker deployment, reverse proxy configuration, automated backups, HTTPS, and admin panel hardening.

## Why Self-Host Vaultwarden?

**Advantages:**
- No third-party access to encrypted data
- Full control over updates and deployment
- Cheap ($5-15/month cloud hosting)
- Works offline if you proxy locally
- Open-source code (audit it)

**Tradeoffs:**
- You manage backups, updates, and security
- No company customer support
- Single point of failure (if your server fails, you manage recovery)
- Small ecosystem (community-maintained, not commercial)

**Best for:** Developers, privacy-conscious individuals, teams wanting complete control. Not for non-technical users.

## Prerequisites

- VPS with 2GB RAM, 20GB storage (Vultr, Hetzner, DigitalOcean)
- Domain name pointing to your VPS IP
- SSH access to VPS
- Basic Linux command-line comfort

**Estimated cost:** $5/month hosting + $10/year domain = $70/year total

## Step 1: Set Up VPS and SSH

Using DigitalOcean as example (Ubuntu 24.04):

```bash
# SSH into VPS
ssh root@your.vps.ip

# Update system
apt update && apt upgrade -y

# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Verify Docker installed
docker --version
```

## Step 2: Create Vaultwarden Docker Compose

Create `/root/vaultwarden/docker-compose.yml`:

```yaml
version: '3'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      DOMAIN: "https://vault.yourdomain.com"
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ORG_ALLOW_USER: "false"
      SEND_FOLDER: "true"
      EXTENDED_LOGGING: "true"
      LOG_FILE: "/data/vaultwarden.log"
      # Admin panel access token (generate: openssl rand -base64 48)
      ADMIN_TOKEN: "your-secret-admin-token-here"
      # SMTP for email invitations (optional, see below)
      MAIL_ENABLED: "false"
    volumes:
      - ./data:/data
      - ./certs:/certs
    networks:
      - vaultwarden-network

  # Caddy reverse proxy with automatic HTTPS
  caddy:
    image: caddy:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      DOMAIN: "vault.yourdomain.com"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data/caddy_data:/data
      - ./data/caddy_config:/config
    networks:
      - vaultwarden-network
    depends_on:
      - vaultwarden

networks:
  vaultwarden-network:
    driver: bridge
```

## Step 3: Configure Reverse Proxy (Caddy)

Create `/root/vaultwarden/Caddyfile`:

```
vault.yourdomain.com {
  encode gzip

  # Reverse proxy to Vaultwarden container
  reverse_proxy localhost:80 {
    # Forward real IP
    header_upstream X-Real-IP {http.request.remote.host}
    header_upstream X-Forwarded-For {http.request.remote.host}
    header_upstream X-Forwarded-Proto {http.request.scheme}
  }

  # Rate limiting (prevent brute force)
  rate_limit 10r/s

  # Security headers
  header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
  header X-Content-Type-Options "nosniff"
  header X-Frame-Options "DENY"
  header X-XSS-Protection "1; mode=block"
  header Referrer-Policy "no-referrer"
  header Content-Security-Policy "default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self'; font-src 'self'"

  # TLS certificate auto-renewal
  tls internal
}
```

## Step 4: Launch Vaultwarden

```bash
cd /root/vaultwarden

# Generate secure admin token
openssl rand -base64 48

# Update docker-compose.yml with the token you just generated
# Edit ADMIN_TOKEN in docker-compose.yml

# Create data directory
mkdir -p data

# Start containers
docker compose up -d

# Verify running
docker compose ps
docker compose logs -f vaultwarden
```

Expected output:
```
vaultwarden | [INFO] Vaultwarden is ready
vaultwarden | Rocket has launched from http://0.0.0.0:80
caddy | caddy started successfully
```

## Step 5: Access Admin Panel

Open browser to `https://vault.yourdomain.com/admin`

You'll see login prompt. Enter your admin token (from Step 3) into the "Master Password Hash" field.

**Admin panel options:**
- Disable new user registrations (already set `SIGNUPS_ALLOWED: false`)
- View user list and delete users
- Monitor server logs
- Update configuration
- Check database stats

**Important:** Disable logins to admin panel after setup:
```bash
# Edit docker-compose.yml
# Add environment variable:
ADMIN_DISABLE_2FA: "true"
ADMIN_SESSION_LIFETIME: 5  # Session expires after 5 minutes
```

Restart:
```bash
docker compose restart vaultwarden
```

## Step 6: Add First User

Two options:

**Option A: Admin panel (easiest)**
1. Go to `vault.yourdomain.com/admin`
2. Click "Users"
3. Click "New User"
4. Enter email, set temporary password
5. Share link with user

**Option B: Manual command (if admin panel fails)**
```bash
docker compose exec -it vaultwarden \
  /vaultwarden hash --preset owasp --input yourpassword
```

## Step 7: Enable SMTP for Email Invitations

Edit `docker-compose.yml`:

```yaml
environment:
  MAIL_ENABLED: "true"
  MAIL_FROM: "vault@yourdomain.com"
  MAIL_FROM_NAME: "Vaultwarden"
  # Example: Gmail with app-specific password
  MAIL_SMTP_HOST: "smtp.gmail.com"
  MAIL_SMTP_SECURITY: "force_tls"
  MAIL_SMTP_PORT: "587"
  MAIL_SMTP_AUTH: "Login"
  MAIL_SMTP_USERNAME: "your-email@gmail.com"
  MAIL_SMTP_PASSWORD: "app-specific-password"
```

**For Gmail:**
1. Enable 2-factor authentication on your Google Account
2. Generate app password: https://myaccount.google.com/apppasswords
3. Use app password above (not your regular password)

Restart:
```bash
docker compose restart vaultwarden
```

## Step 8: Automated Backups

Create backup script `/root/vaultwarden/backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/root/vaultwarden/backups"
DB_PATH="/root/vaultwarden/data/db.sqlite3"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
cp $DB_PATH $BACKUP_DIR/db_$TIMESTAMP.sqlite3

# Backup attachments and sends
tar -czf $BACKUP_DIR/vaultwarden_$TIMESTAMP.tar.gz \
  /root/vaultwarden/data/attachments \
  /root/vaultwarden/data/sends \
  /root/vaultwarden/data/icon_cache

# Delete backups older than 30 days
find $BACKUP_DIR -mtime +30 -delete

# Upload to remote storage (optional)
# aws s3 cp $BACKUP_DIR/db_$TIMESTAMP.sqlite3 s3://your-bucket/backups/
```

Make executable and add to cron:

```bash
chmod +x /root/vaultwarden/backup.sh

# Edit crontab
crontab -e

# Add line (daily backup at 2am):
0 2 * * * /root/vaultwarden/backup.sh
```

Verify:
```bash
# Run backup manually
/root/vaultwarden/backup.sh

# Check results
ls -la /root/vaultwarden/backups/
```

## Step 9: Enable Two-Factor Authentication (2FA)

In admin panel:
1. Settings → Authentication
2. Set `TWOFACTOR` options:
 - `TWOFACTOR_EMAIL`: Allow email-based 2FA
 - `TWOFACTOR_AUTHENTICATOR`: Allow authenticator app (Google Authenticator, Authy)

```yaml
environment:
  TWOFACTOR: "true"
  TWOFACTOR_EMAIL: "true"
  TWOFACTOR_AUTHENTICATOR: "true"
  TWOFACTOR_REMEMBER: "true"
  TWOFACTOR_REMEMBER_EXPIRES_IN: 30
```

## Step 10: Configure Organization (Optional)

If you want to share passwords with family/team:

1. Log in to web vault: `vault.yourdomain.com`
2. Click "New Organization"
3. Name it ("Family Passwords" or "Work Team")
4. Add users via admin panel or send invitations
5. Create collections (folders) and assign to users

**Example org structure:**
```
Organization: MyFamily
├─ Collection: Banking (shared with spouse)
├─ Collection: Streaming (shared with all)
└─ Collection: Personal (only me)
```

## Client Setup

**Desktop (Windows, Mac, Linux):**
1. Download Bitwarden client (works with Vaultwarden)
2. Login with your email
3. Settings → Server URL → `https://vault.yourdomain.com`
4. Restart client, re-authenticate

**Mobile (iOS, Android):**
1. Download Bitwarden app
2. Settings → Server URL → `https://vault.yourdomain.com`
3. Login with email
4. Pin app to home screen

**Browser Extension:**
1. Install Bitwarden extension (Chrome, Firefox, Safari, Edge)
2. Click icon → Settings → Server URL → `https://vault.yourdomain.com`
3. Login once
4. Auto-fill works normally

## Monitoring and Maintenance

**Weekly:**
- Check logs: `docker compose logs vaultwarden`
- Look for errors (bad logins, sync failures)

**Monthly:**
- Verify backups exist and are fresh
- Test restore procedure (copy backup, start test container)
- Check for Vaultwarden updates:
 ```bash
  docker compose pull
  docker compose up -d
  ```

**Quarterly:**
- Audit users (admin panel → Users)
- Remove inactive accounts
- Review security logs

## Troubleshooting

**Connection refused when accessing admin panel:**
```bash
# Check if containers running
docker compose ps

# Check logs
docker compose logs caddy
docker compose logs vaultwarden

# Restart all
docker compose restart
```

**Email not sending:**
```bash
# Test SMTP connection
docker compose exec -it vaultwarden \
  /vaultwarden test-mail your-email@gmail.com
```

**Database locked error:**
```bash
# Stop containers, wait 10 seconds, restart
docker compose down
sleep 10
docker compose up -d
```

**Forgotten admin password:**
You can't reset it directly. Instead:
1. Stop Vaultwarden
2. Replace database with backup
3. Restart

This is why backups are critical.

## Security Hardening Checklist

- [ ] Changed default admin token (not empty string)
- [ ] Disabled public signup (`SIGNUPS_ALLOWED: false`)
- [ ] Enabled HTTPS with valid certificate
- [ ] Configured rate limiting in Caddy
- [ ] Set strong master password (20+ chars)
- [ ] Enabled 2FA for your account
- [ ] Scheduled daily backups
- [ ] Stored backups on external drive or cloud
- [ ] Reviewed firewall rules (only ports 80/443 open)
- [ ] Tested backup restoration

## Cost Breakdown

**Annual cost:**
- VPS: $60/year (DigitalOcean $5/month)
- Domain: $10/year (Google Domains)
- Total: **$70/year**

**vs. Bitwarden Premium:**
- Bitwarden: $10/month = $120/year
- Savings: $50/year

**but you pay in time:**
- Initial setup: 2-3 hours
- Monthly maintenance: 1-2 hours
- Backup restore (disaster): 1-2 hours

## Alternatives

**Bitwarden Cloud ($10/month):**
- Pro: No maintenance, free mobile apps, family sharing
- Con: Trust Bitwarden, pay subscription

**1Password ($36-99/year):**
- Pro: Slick UX, family plan
- Con: Proprietary, more expensive

**KeePass (free, local):**
- Pro: Zero cloud trust
- Con: No sync, no mobile, management burden

## Related Articles

- [Self-Hosted Password Manager Comparison](self-hosted-password-manager-comparison)
- [Bitwarden vs Vaultwarden Self-Hosted: A Technical Comparison](/bitwarden-vs-vaultwarden-self-hosted-comparison/)
- [How To Set Up Beneficiary Access For Cloud Password Manager](/how-to-set-up-beneficiary-access-for-cloud-password-manager-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
