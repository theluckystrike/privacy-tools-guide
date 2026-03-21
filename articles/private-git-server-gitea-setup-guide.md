---
layout: default
title: "Self-Hosted Private Git Server with Gitea"
description: "Set up a private Gitea git server on a VPS or home server with HTTPS, SSH key auth, and access controls to keep your code off GitHub and GitLab's servers"
date: 2026-03-21
author: theluckystrike
permalink: /private-git-server-gitea-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

GitHub and GitLab are convenient but they hold your code. GitHub (owned by Microsoft) scans repositories for content policy violations, sells data for Copilot training, and is subject to US legal requests. If your code contains unreleased work, personal projects, sensitive configurations, or anything you'd prefer to keep private, a self-hosted git server gives you complete control.

Gitea is a lightweight, fast git service written in Go. It runs on minimal hardware — a Raspberry Pi or a $5/month VPS handles dozens of repositories and users comfortably.

## Prerequisites

- A Linux server (Debian/Ubuntu 22.04+ recommended) with a public IP
- A domain name pointing to the server (for HTTPS)
- Open ports: 22 (SSH), 80 (HTTP, for cert renewal), 443 (HTTPS), and optionally 3000 (if not using reverse proxy)

## Step 1: Install Gitea

```bash
# Download the latest Gitea binary
# Check https://dl.gitea.com/gitea/ for the latest version
GITEA_VERSION="1.22.0"
wget "https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64"

# Verify the checksum
wget "https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64.sha256"
sha256sum -c "gitea-${GITEA_VERSION}-linux-amd64.sha256"

# Install
sudo mv "gitea-${GITEA_VERSION}-linux-amd64" /usr/local/bin/gitea
sudo chmod 755 /usr/local/bin/gitea

# Create gitea user and directories
sudo adduser --system --shell /bin/bash --gecos 'Gitea' --group \
  --disabled-password --home /home/git git

sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea
```

## Step 2: Configure Gitea as a systemd Service

```bash
sudo tee /etc/systemd/system/gitea.service << 'EOF'
[Unit]
Description=Gitea (Git with a cup of tea)
After=network.target
After=postgresql.service mysql.service mariadb.service

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
ProtectSystem=true
ExecStartPre=/bin/bash -c 'while ! pg_isready; do sleep 1; done'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gitea
sudo systemctl start gitea
```

## Step 3: Set Up a Reverse Proxy with Nginx and HTTPS

Install Nginx and Certbot:

```bash
sudo apt install nginx certbot python3-certbot-nginx

# Create Nginx config for Gitea
sudo tee /etc/nginx/sites-available/gitea << 'EOF'
server {
    listen 80;
    server_name git.yourdomain.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name git.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/git.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/git.yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
        client_max_body_size 512m;
        proxy_read_timeout 300;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/
sudo nginx -t

# Obtain TLS certificate
sudo certbot --nginx -d git.yourdomain.com

# Enable Nginx
sudo systemctl enable nginx
sudo systemctl reload nginx
```

## Step 4: Initial Gitea Setup

Browse to `https://git.yourdomain.com` for the setup wizard.

Key settings to configure:

**Database**: SQLite (simple, fine for personal use) or PostgreSQL (better for multiple users)

**General Settings**:
- Site Title: your name (not publicly readable to crawlers if you disable registration)
- Repository Root Path: `/var/lib/gitea/data/repositories`
- Git Hooks Path: `/var/lib/gitea/data`
- Base URL: `https://git.yourdomain.com`

**Server and Third-Party Service Settings**:
- Disable Gravatar (this sends email hashes to Gravatar's CDN — a privacy leak)
- Uncheck "Enable Federated Avatars Lookup"

**Admin Account**: Create your admin account here. Use a strong passphrase.

## Step 5: Harden Gitea Configuration

Edit `/etc/gitea/app.ini` after initial setup:

```ini
[server]
PROTOCOL = http
HTTP_PORT = 3000
ROOT_URL = https://git.yourdomain.com
DISABLE_SSH = false
SSH_PORT = 22
START_SSH_SERVER = false

[service]
; Disable public registration — only invite known users
DISABLE_REGISTRATION = true
; Require admin approval for new users (if registration enabled)
REGISTER_MANUAL_CONFIRM = true
; Disable avatar services
ENABLE_GRAVATAR = false
DISABLE_GRAVATAR = true
ENABLE_FEDERATED_AVATAR = false

[repository]
; Default: make new repos private
DEFAULT_PRIVATE = true
; Disable mirroring from external sources if not needed
; DISABLE_MIRRORS = true

[log]
MODE = file
LEVEL = Warn
ROOT_PATH = /var/lib/gitea/log

[security]
; Generate a random secret key during install (already set)
; Minimum password length
MIN_PASSWORD_LENGTH = 12
PASSWORD_COMPLEXITY = lower,upper,digit,spec
; Force 2FA for admin accounts
TWO_FACTOR_REQUIRED_GROUPS = admins
```

Apply changes:

```bash
sudo systemctl restart gitea
```

## Step 6: Enable Two-Factor Authentication

In Gitea web interface:
1. Click your profile → Settings → Security
2. Enable TOTP two-factor authentication with your authenticator app
3. Save the recovery codes securely

For all users, enforce 2FA via the admin panel:
- Admin Panel → Users → [user] → Edit → Require 2FA: checked

## Step 7: Configure SSH Access

Add your SSH public key for command-line git operations:

1. In Gitea: Profile → Settings → SSH/GPG Keys → Add Key
2. Paste your public key (`~/.ssh/id_ed25519.pub`)

Clone using SSH:
```bash
git clone git@git.yourdomain.com:yourusername/your-repo.git
```

Gitea's SSH server runs through the system SSH daemon using authorized keys. The `git` system user handles authentication.

## Step 8: Set Up Automated Backups

```bash
#!/bin/bash
# /usr/local/bin/gitea-backup
BACKUP_DIR="/var/backups/gitea"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

# Gitea built-in backup
sudo -u git gitea dump -c /etc/gitea/app.ini \
  --file "$BACKUP_DIR/gitea-dump-$TIMESTAMP.zip"

# Keep only last 7 backups
find "$BACKUP_DIR" -name "gitea-dump-*.zip" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/gitea-dump-$TIMESTAMP.zip"
```

```bash
chmod +x /usr/local/bin/gitea-backup
echo "0 3 * * * root /usr/local/bin/gitea-backup" | sudo tee /etc/cron.d/gitea-backup
```

## Migrating from GitHub/GitLab

Gitea can mirror or import existing repositories:

```bash
# Via CLI
git clone --mirror https://github.com/user/repo.git
cd repo.git
git remote set-url origin git@git.yourdomain.com:user/repo.git
git push --mirror

# Via Gitea web UI
# + (new repo) → Migrate → GitHub/GitLab/Bitbucket
# Enter credentials and select repos to migrate
```

## Related Reading

- [How to Set Up a Privacy-Focused Home Server](/how-to-set-up-secure-home-server-for-self-hosting-privacy-tools/)
- [SSH Server Hardening Guide](/ssh-server-hardening-guide/)
- [Encrypt Cloud Storage with Rclone Before Uploading](/secure-cloud-storage-encryption-rclone/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
