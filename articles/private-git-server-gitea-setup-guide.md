---
layout: default
title: "Self-Hosted Private Git Server with Gitea"
description: "Set up a private Gitea git server on a VPS or home server with HTTPS, SSH key auth, and access controls to keep your code off GitHub and GitLab's servers"
date: 2026-03-21
author: theluckystrike
permalink: /private-git-server-gitea-setup-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

GitHub and GitLab are convenient but they hold your code. GitHub (owned by Microsoft) scans repositories for content policy violations, sells data for Copilot training, and is subject to US legal requests. If your code contains unreleased work, personal projects, sensitive configurations, or anything you'd prefer to keep private, a self-hosted git server gives you complete control.

Gitea is a lightweight, fast git service written in Go. It runs on minimal hardware. a Raspberry Pi or a $5/month VPS handles dozens of repositories and users comfortably.

Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Install Gitea](#step-1-install-gitea)
- [Step 2: Configure Gitea as a systemd Service](#step-2-configure-gitea-as-a-systemd-service)
- [Step 3: Set Up a Reverse Proxy with Nginx and HTTPS](#step-3-set-up-a-reverse-proxy-with-nginx-and-https)
- [Step 4: Initial Gitea Setup](#step-4-initial-gitea-setup)
- [Step 5: Harden Gitea Configuration](#step-5-harden-gitea-configuration)
- [Step 6: Enable Two-Factor Authentication](#step-6-enable-two-factor-authentication)
- [Step 7: Configure SSH Access](#step-7-configure-ssh-access)
- [Step 8: Set Up Automated Backups](#step-8-set-up-automated-backups)
- [Setting Up Gitea Webhooks for Deployment](#setting-up-gitea-webhooks-for-deployment)
- [Using Gitea Actions for CI/CD](#using-gitea-actions-for-cicd)
- [Access Control and Team Management](#access-control-and-team-management)
- [Migrating from GitHub/GitLab](#migrating-from-githubgitlab)
- [Related Reading](#related-reading)

Prerequisites

- A Linux server (Debian/Ubuntu 22.04+ recommended) with a public IP
- A domain name pointing to the server (for HTTPS)
- Open ports: 22 (SSH), 80 (HTTP, for cert renewal), 443 (HTTPS), and optionally 3000 (if not using reverse proxy)

Step 1: Install Gitea

```bash
Download the latest Gitea binary
Check https://dl.gitea.com/gitea/ for the latest version
GITEA_VERSION="1.22.0"
wget "https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64"

Verify the checksum
wget "https://dl.gitea.com/gitea/${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64.sha256"
sha256sum -c "gitea-${GITEA_VERSION}-linux-amd64.sha256"

Install
sudo mv "gitea-${GITEA_VERSION}-linux-amd64" /usr/local/bin/gitea
sudo chmod 755 /usr/local/bin/gitea

Create gitea user and directories
sudo adduser --system --shell /bin/bash --gecos 'Gitea' --group \
  --disabled-password --home /home/git git

sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea
```

Step 2: Configure Gitea as a systemd Service

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

Step 3: Set Up a Reverse Proxy with Nginx and HTTPS

Install Nginx and Certbot:

```bash
sudo apt install nginx certbot python3-certbot-nginx

Create Nginx config for Gitea
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

Obtain TLS certificate
sudo certbot --nginx -d git.yourdomain.com

Enable Nginx
sudo systemctl enable nginx
sudo systemctl reload nginx
```

Step 4: Initial Gitea Setup

Browse to `https://git.yourdomain.com` for the setup wizard.

Key settings to configure:

Database: SQLite (simple, fine for personal use) or PostgreSQL (better for multiple users)

General Settings:
- Site Title: your name (not publicly readable to crawlers if you disable registration)
- Repository Root Path: `/var/lib/gitea/data/repositories`
- Git Hooks Path: `/var/lib/gitea/data`
- Base URL: `https://git.yourdomain.com`

Server and Third-Party Service Settings:
- Disable Gravatar (this sends email hashes to Gravatar's CDN. a privacy leak)
- Uncheck "Enable Federated Avatars Lookup"

Admin Account: Create your admin account here. Use a strong passphrase.

Step 5: Harden Gitea Configuration

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
; Disable public registration. only invite known users
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

Step 6: Enable Two-Factor Authentication

In Gitea web interface:
1. Click your profile → Settings → Security
2. Enable TOTP two-factor authentication with your authenticator app
3. Save the recovery codes securely

For all users, enforce 2FA via the admin panel:
- Admin Panel → Users → [user] → Edit → Require 2FA: checked

Step 7: Configure SSH Access

Add your SSH public key for command-line git operations:

1. In Gitea: Profile → Settings → SSH/GPG Keys → Add Key
2. Paste your public key (`~/.ssh/id_ed25519.pub`)

Clone using SSH:
```bash
git clone git@git.yourdomain.com:yourusername/your-repo.git
```

Gitea's SSH server runs through the system SSH daemon using authorized keys. The `git` system user handles authentication.

Hardening SSH on the Server

Your git server is a public-facing machine. Reduce its SSH attack surface:

```bash
/etc/ssh/sshd_config. key changes
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
AllowUsers git youradminuser
MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
```

Apply the changes:

```bash
sudo sshd -t           # test config before reloading
sudo systemctl reload ssh
```

If you change the SSH port away from 22, update your Gitea `app.ini`:

```ini
[server]
SSH_PORT = 2222
```

And update your git remote URLs accordingly:

```bash
git remote set-url origin ssh://git@git.yourdomain.com:2222/user/repo.git
```

Step 8: Set Up Automated Backups

```bash
#!/bin/bash
/usr/local/bin/gitea-backup
BACKUP_DIR="/var/backups/gitea"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

Gitea built-in backup
sudo -u git gitea dump -c /etc/gitea/app.ini \
  --file "$BACKUP_DIR/gitea-dump-$TIMESTAMP.zip"

Keep only last 7 backups
find "$BACKUP_DIR" -name "gitea-dump-*.zip" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/gitea-dump-$TIMESTAMP.zip"
```

```bash
chmod +x /usr/local/bin/gitea-backup
echo "0 3 * * * root /usr/local/bin/gitea-backup" | sudo tee /etc/cron.d/gitea-backup
```

Offsite Backup with rclone

Keeping backups on the same server provides no protection against hardware failure or server compromise. Offload to encrypted remote storage:

```bash
Install rclone and configure a remote (Backblaze B2, S3, etc.)
curl https://rclone.org/install.sh | sudo bash
rclone config  # follow prompts for your provider

Add to the backup script, after the dump
rclone copy "$BACKUP_DIR/gitea-dump-$TIMESTAMP.zip" remote:gitea-backups/ \
  --b2-chunk-size 96M
```

For maximum privacy, encrypt the backup before uploading using rclone's built-in crypt backend. This means even your cloud storage provider cannot read the backup contents.

Setting Up Gitea Webhooks for Deployment

Gitea supports webhooks that trigger on push, pull request, and release events. Use them to kick off deployments or notify external services without exposing your code:

1. Go to your repository → Settings → Webhooks → Add Webhook
2. Select Gitea as the type
3. Set the Payload URL to your deployment endpoint (e.g., `https://deploy.yourdomain.com/hook`)
4. Choose application/json as the content type
5. Set a secret token. Gitea signs each payload with HMAC-SHA256 using this token

On your deployment server, verify the signature before acting:

```bash
#!/bin/bash
verify-webhook.sh
SIGNATURE="$HTTP_X_GITEA_SIGNATURE"
BODY=$(cat /dev/stdin)
EXPECTED=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $2}')

if [ "$SIGNATURE" != "sha256=$EXPECTED" ]; then
  echo "Signature mismatch. ignoring request" >&2
  exit 1
fi

Deploy
git -C /var/www/myapp pull origin main
systemctl restart myapp
```

This pattern lets you implement continuous deployment entirely within your own infrastructure, with no dependency on external CI services.

Using Gitea Actions for CI/CD

Gitea 1.21+ includes a built-in CI/CD system compatible with GitHub Actions syntax. You can run your own runners without sending code to external services:

```bash
Download the Gitea runner binary
RUNNER_VERSION="0.2.6"
wget "https://dl.gitea.com/act_runner/${RUNNER_VERSION}/act_runner-${RUNNER_VERSION}-linux-amd64"
sudo mv "act_runner-${RUNNER_VERSION}-linux-amd64" /usr/local/bin/act_runner
sudo chmod 755 /usr/local/bin/act_runner

Register the runner with your instance
act_runner register \
  --instance https://git.yourdomain.com \
  --token YOUR_RUNNER_TOKEN \
  --name my-runner \
  --labels ubuntu-latest:docker://node:20-bullseye
```

Create a workflow file in any repository at `.gitea/workflows/test.yml`:

```yaml
name: Run Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          npm install
          npm test
```

This gives you a fully self-hosted CI pipeline. Your code, build artifacts, and test results never leave your infrastructure.

Access Control and Team Management

Gitea's organization and team system gives granular control over who can read or write which repositories:

- Organizations group repositories under a shared namespace (e.g., `git.yourdomain.com/myteam/repo`)
- Teams within an organization hold members and define their permission level: Owner, Admin, Write, Read, or None
- Repository visibility can be set to Public, Internal (organization members only), or Private per-repo

For a solo developer or small team, the simplest approach is to create one organization, add all your repositories under it, and add collaborators as members of a Write-permission team. They get access to everything the team covers without requiring per-repo invitations.

If you run Gitea for a development team with contractors or external contributors:

1. Admin Panel → Organizations → New Organization
2. Create teams: `core-devs` (Write), `contractors` (Read on specific repos), `ci-runners` (Read only)
3. Add repositories to each team from the team's page
4. Invite members by email or username

This model keeps sensitive repositories invisible to contractors while sharing only the repositories they need.

Migrating from GitHub/GitLab

Gitea can mirror or import existing repositories:

```bash
Via CLI
git clone --mirror https://github.com/user/repo.git
cd repo.git
git remote set-url origin git@git.yourdomain.com:user/repo.git
git push --mirror

Via Gitea web UI
+ (new repo) → Migrate → GitHub/GitLab/Bitbucket
Enter credentials and select repos to migrate
```

Related Articles

- [How To Configure Openpgp Key Server For Organization](/how-to-configure-openpgp-key-server-for-organization-interna/)
- [Mumble Encrypted Voice Chat Server Setup For Private Team](/mumble-encrypted-voice-chat-server-setup-for-private-team-co/)
- [Set Up Mail In A Box Private Email Server Complete 2026](/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Self-Hosted Private Video Calling Setup Guide](/private-video-calling-selfhosted-guide/)
- [SSH Server Hardening Guide](/ssh-server-hardening-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

Can I trust these tools with sensitive data?

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

{% endraw %}
