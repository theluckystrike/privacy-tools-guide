---
layout: default
title: "Cryptpad Encrypted Collaboration Suite Self Hosting Setup"
description: "Learn how to self-host CryptPad for end-to-end encrypted document collaboration. This practical guide covers Docker deployment, configuration, security"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /cryptpad-encrypted-collaboration-suite-self-hosting-setup-gu/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, collaboration]
---

{% raw %}

CryptPad is an end-to-end encrypted collaboration suite that provides real-time document editing, spreadsheets, presentations, and more without sacrificing privacy. Unlike conventional office suites, CryptPad encrypts all data on the client side, your server never sees plaintext content. Self-hosting gives teams full control over their data while maintaining CryptPad's zero-knowledge architecture.

This guide walks through deploying CryptPad using Docker, configuring essential settings, hardening security, and managing team access.

Prerequisites

Before starting, ensure you have:

- A Linux server with at least 2GB RAM and 20GB storage
- Docker and Docker Compose installed
- A domain pointing to your server (for TLS certificates)
- Basic familiarity with the command line

Step 1: Install CryptPad with Docker

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
Add to docker-compose.yml
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

Step 2: Understand CryptPad's Encryption Model

CryptPad employs a unique encryption architecture worth understanding before production deployment. Each document generates a random 256-bit key locally. The URL itself contains the encryption key, sharing a document URL grants decryption access. This design means your server stores only encrypted blobs; even with full server access, administrators cannot read user content.

User accounts use a different mechanism. When users create accounts, CryptPad derives encryption keys from their password using Argon2id. The server stores authentication credentials separately from encryption keys. This separation ensures that compromising the database does not automatically expose document contents.

Document types in CryptPad include CryptDrive (file manager), Pad (rich text), Code (code editor), Slide (presentations), Sheet (spreadsheets), Kanban, and Form. Each document type shares the same zero-knowledge guarantee. When a user opens a shared document link, the decryption key in the URL fragment never gets sent to the server because browsers do not transmit the fragment portion of a URL in HTTP requests.

Step 3: Configuration and Customization

CryptPad stores configuration in `/var/cryptpad/customize` when using Docker. Key files include:

Limit Registration:

```bash
echo '{"limitRegistration": true}' > customize/config.json
```

This prevents public signup. Combine with admin invite codes for controlled deployments.

Admin Mode:

To access admin features, create an admin user account, then run:

```bash
docker exec cryptpad node bin/admin.js addadmin user@yourdomain.com
```

Admin users can create invite codes, manage storage quotas, and access usage statistics.

Storage Quotas:

Set per-user storage limits in `customize/config.json`:

```json
{
  "limitRegistration": true,
  "maxUploadSize": 52428800,
  "defaultStorage": 104857600
}
```

This limits uploads to 50MB and grants each user 100MB by default.

Custom Branding:

For teams deploying CryptPad internally, you can replace the default logo and color scheme. Copy the customize template:

```bash
docker exec cryptpad cp -r /var/cryptpad/customize.dist/. /var/cryptpad/customize/
```

Then edit `customize/application_config.js` to set your organization name and logo path. The `customize/pages.js` file controls landing page content. These changes survive container updates because the customize directory is mounted as a volume.

Step 4: Security Hardening

Production deployments require attention to several security areas:

Disable Remote Logging:

By default, CryptPad may send error reports. Disable this in production:

```bash
echo '{"sentry": false}' > customize/config.json
```

Configure Content Security Policy:

Add custom CSP headers through your reverse proxy. For nginx:

```
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
```

Database Encryption:

While CryptPad encrypts document content, user credentials and metadata remain. Consider enabling disk encryption at the filesystem level:

```bash
For Ubuntu with LUKS
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 cryptpad_data
sudo mkfs.ext4 /dev/mapper/cryptpad_data
```

Mount the encrypted volume at `/var/lib/docker/volumes/cryptpad_data` before starting containers.

Network Isolation:

Run CryptPad in a dedicated network namespace:

```bash
docker network create --driver bridge cryptpad_net
Add to docker-compose.yml networks section
networks:
  - cryptpad_net
```

Rate Limiting Nginx:

If you front CryptPad with nginx instead of Caddy, add rate limiting to prevent abuse:

```nginx
limit_req_zone $binary_remote_addr zone=cryptpad_api:10m rate=30r/m;

server {
    location /api/ {
        limit_req zone=cryptpad_api burst=10 nodelay;
        proxy_pass http://cryptpad:3000;
    }
}
```

This blocks brute-force login attempts and API abuse without affecting normal use patterns.

Step 5: Team User Management

Self-hosted CryptPad offers several user management strategies:

Invite Codes:

Generate invite codes for controlled registration:

```bash
docker exec cryptpad node bin/admin.js newinvite --count 10 --expiry 30d
```

This creates 10 codes valid for 30 days. Distribute codes to team members.

LDAP Integration:

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

Storage Management:

Monitor storage with admin scripts:

```bash
List users and storage usage
docker exec cryptpad node bin/admin.js listusers

Set specific user quota
docker exec cryptpad node bin/admin.js setquota user@domain.com 209715200
```

Teams Feature:

CryptPad's Teams functionality lets groups share a common drive with role-based permissions. Create a team from within the web interface by clicking your avatar and selecting "New Team." Teams have separate storage quotas from individual users and support Owner, Admin, Member, and Viewer roles. This makes CryptPad suitable for department-level document management without exposing individual drives.

Step 6: Backup and Recovery

Implement regular backups of the CryptPad data volume:

```bash
#!/bin/bash
backup-cryptpad.sh
DATE=$(date +%Y%m%d)
docker run --rm -v cryptpad_data:/data -v $(pwd):/backup alpine tar czf /backup/cryptpad-backup-$DATE.tar.gz /data
```

Schedule daily backups with cron:

```bash
0 2 * * * /path/to/backup-cryptpad.sh
```

Test restoration periodically:

```bash
Stop container
docker-compose down
Restore
docker run --rm -v cryptpad_data:/data -v $(pwd):/backup alpine tar xzf /backup/cryptpad-backup-20260316.tar.gz -C /
Start container
docker-compose up -d
```

For off-site backups, pipe the tarball through `rclone` to push it to a S3-compatible bucket or Backblaze B2:

```bash
docker run --rm -v cryptpad_data:/data alpine tar czf - /data | \
  rclone rcat s3:your-bucket/cryptpad-backup-$(date +%Y%m%d).tar.gz
```

This keeps your encrypted CryptPad data off-site without requiring decryption at rest on the backup host.

Performance Considerations

For teams exceeding 50 users, adjust these settings in `customize/config.json`:

```json
{
  "worker": true,
  "httpUnsafeOrigin": "https://cryptpad.yourdomain.com",
  "adminEmail": "admin@yourdomain.com"
}
```

Consider enabling Redis for session management in high-load scenarios by adding a Redis service to your compose file and configuring `customize/redis.json`.

For very active teams, monitor resource usage with:

```bash
docker stats cryptpad --no-stream
```

If CPU usage spikes during peak hours, increase the Node.js worker pool by setting `UV_THREADPOOL_SIZE=16` in your compose environment block. Memory pressure usually indicates too many concurrent document sessions; raising your server to 4GB RAM resolves most scaling issues up to 200 concurrent users.

Step 7: Updating CryptPad

Pull the latest image and restart to apply updates:

```bash
docker-compose pull
docker-compose up -d
```

CryptPad releases follow semantic versioning. Check the [CryptPad changelog](https://github.com/cryptpad/cryptpad/blob/main/CHANGELOG.md) before major version upgrades, as some releases require running migration scripts:

```bash
docker exec cryptpad node scripts/migrate.js
```

Run migrations with the container stopped from serving external traffic to avoid data corruption during schema changes.

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to complete this setup?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Encrypted Collaboration Tools For Remote Teams That Respect](/encrypted-collaboration-tools-for-remote-teams-that-respect-/)
- [Secure Document Collaboration Alternatives to Google](/secure-document-collaboration-alternatives-to-google-docs-wi/)
- [Privacy Setup For Psychologist Telehealth Sessions Encrypted](/privacy-setup-for-psychologist-telehealth-sessions-encrypted/)
- [Encrypted File Sync for Teams Comparison: A Developer Guide](/encrypted-file-sync-for-teams-comparison/)
- [Nextcloud End to End Encryption Setup Guide](/nextcloud-end-to-end-encryption-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
