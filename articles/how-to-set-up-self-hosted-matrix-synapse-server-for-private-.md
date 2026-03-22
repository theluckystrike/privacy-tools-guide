---
layout: default
title: "How To Set Up Self Hosted Matrix Synapse Server For Private"
description: "Matrix is an open protocol for real-time communication, and Synapse is the reference implementation of a Matrix homeserver. Running your own Synapse instance"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-self-hosted-matrix-synapse-server-for-private-/
categories: [guides, security]
reviewed: true
score: 9
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

Add security headers to all responses to prevent clickjacking and content-type sniffing:

```nginx
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
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

For mobile users, Element is available on Android via F-Droid (preferred for privacy) and both iOS and Android through their respective app stores. The SchildiChat client on Android provides additional privacy-focused defaults including automatic room key backup and improved notification handling.

## Enabling End-to-End Encryption

Matrix supports end-to-end encryption (E2EE) by default using the Olm and Megolm protocols. When you create a new room in Element, encryption is available by toggling the encryption setting in room settings. Each device gets its own encryption keys stored locally.

For additional security, verify key fingerprints in the settings under "Security & Privacy". This ensures you're communicating with the correct recipients and not victim to man-in-the-middle attacks.

To enforce encryption organization-wide and prevent accidentally creating unencrypted rooms, configure Synapse's room defaults in `homeserver.yaml`:

```yaml
encryption_enabled_by_default_for_room_presets:
  private_chat: true
  trusted_private_chat: true
  public_chat: false
```

This ensures any private or trusted private room created on your server defaults to encrypted. Public rooms remain unencrypted since encryption in open rooms provides limited privacy benefit—anyone can join and read messages.

### Cross-Signing for Multi-Device Verification

Matrix's cross-signing system allows you to verify all your devices with a single action rather than verifying each device pair individually. After logging in on a second device, go to **Settings → Security & Privacy → Cross-signing** and verify the new session from an already-verified device. Once cross-signing is set up, new room members appear as verified automatically if they share a verified identity across devices.

## Hardening Your Synapse Server

Several configuration changes improve your server's security. Edit the `homeserver.yaml` file in your data directory:

### Disable Public Registration

Prevent unauthorized users from creating accounts:

```yaml
enable_registration: false
```

When you need to add new members, generate single-use registration tokens:

```bash
docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml \
  --generate-registration-token http://localhost:8008
```

Share the token over a separate encrypted channel. Tokens expire after one use, preventing unauthorized account creation even if a token leaks.

### Rate Limiting

Synapse includes configurable rate limiting to prevent brute-force login attempts and message flooding:

```yaml
rc_login:
  address:
    per_second: 0.15
    burst_count: 5
  account:
    per_second: 0.18
    burst_count: 4
  failed_attempts:
    per_second: 0.19
    burst_count: 7
```

Tight rate limits significantly raise the cost of credential-stuffing attacks while having no practical impact on legitimate users.

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

For PostgreSQL deployments, use `pg_dump` with encryption:

```bash
#!/bin/bash
pg_dump -U synapse synapse | gpg --symmetric --cipher-algo AES256 \
  -o /backup/synapse-$(date +%Y%m%d).sql.gpg
```

### Update Synapse Regularly

Stay current with security patches:

```bash
docker-compose pull
docker-compose up -d
```

Subscribe to the Matrix Security Disclosure mailing list at matrix.org to receive notifications of new CVEs before they are publicly disclosed.

## Monitoring and Log Management

For a privacy-focused deployment, log retention deserves deliberate attention. Synapse logs contain IP addresses and user identifiers by default. Configure log rotation and minimal retention:

```yaml
# homeserver.yaml
log_config: "/data/log.yaml"
```

In your `/data/log.yaml`:

```yaml
version: 1
formatters:
  precise:
    format: '%(asctime)s - %(name)s - %(lineno)d - %(levelname)s - %(message)s'
handlers:
  file:
    class: logging.handlers.TimedRotatingFileHandler
    filename: /data/homeserver.log
    when: midnight
    backupCount: 3     # Keep only 3 days of logs
    encoding: utf8
    formatter: precise
root:
  level: WARNING       # Reduce verbosity to avoid logging user activity
  handlers: [file]
```

Three days of log retention at WARNING level captures operational errors without accumulating a detailed record of user sessions. If your server is subpoenaed, there is little to hand over.

## Configuring a Tor Onion Service for Hidden Access

For high-risk deployments where server location must remain unknown, run Synapse behind a Tor onion service. This hides the server's IP address from both users and potential adversaries, and provides a stable address that remains accessible even if the domain is seized or blocked.

Install Tor on the host:

```bash
sudo apt install tor
```

Add to `/etc/tor/torrc`:

```
HiddenServiceDir /var/lib/tor/matrix_hidden/
HiddenServicePort 443 127.0.0.1:8008
```

Restart Tor and retrieve your `.onion` address:

```bash
sudo systemctl restart tor
sudo cat /var/lib/tor/matrix_hidden/hostname
```

Update your `homeserver.yaml` to accept connections at the onion address:

```yaml
public_baseurl: "http://youronionaddress.onion/"
```

Clients that support onion addresses (Element Desktop with Tor configured as a SOCKS5 proxy on 127.0.0.1:9050) can connect directly without exposing the connection to their ISP. Share the `.onion` address only through already-encrypted channels.

## Testing Federation

One of Matrix's strengths is federation—servers communicating with each other. Test federation by creating a room and inviting a user from another Matrix server, such as matrix.org. If federation works, you'll see messages flow between servers.

To verify federation is working, check the server's federation API:

```bash
curl -X GET "https://matrix.yourdomain.com/_matrix/federation/v1/version"
```

A successful response indicates federation is operational.

Use the Matrix Federation Tester at federationtester.matrix.org to diagnose delegation issues, certificate problems, and firewall misconfiguration without leaving any trace on the target server.

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

PostgreSQL dramatically improves performance for servers with more than a handful of active users. SQLite is acceptable for personal use or small teams; beyond roughly ten concurrent users, message delivery latency noticeably degrades.



## Frequently Asked Questions


**How long does it take to set up self hosted matrix synapse server for private?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.


**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.


**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.


**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.


**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.


## Related Articles

- [Self-Hosted Email Server Privacy Comparison](/privacy-tools-guide/self-hosted-email-privacy-comparison/)
- [How To Set Up Jitsi Meet Self Hosted Encrypted Video Confere](/privacy-tools-guide/how-to-set-up-jitsi-meet-self-hosted-encrypted-video-confere/)
- [Set Up a Secure Home Server for Self-Hosting Privacy Tools.](/privacy-tools-guide/how-to-set-up-secure-home-server-for-self-hosting-privacy-tools/)
- [Set Up Mail In A Box Private Email Server Complete 2026](/privacy-tools-guide/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Matrix/Element vs Signal for Private Group Communication](/privacy-tools-guide/matrix-element-vs-signal-for-private-group-communication-comparison/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
