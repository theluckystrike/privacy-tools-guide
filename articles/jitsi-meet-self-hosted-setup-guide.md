---
layout: default

permalink: /jitsi-meet-self-hosted-setup-guide/
description: "Follow this guide to jitsi meet self hosted setup guide with practical examples, tips, and step-by-step instructions for getting the best results."
tags: [privacy-tools-guide]
author: "Privacy Tools Guide"
reviewed: true
score: 8
date: 2026-03-15
categories: [guides]
---

layout: default
title: "Jitsi Meet Self Hosted Setup Guide"
description: "Deploy a self-hosted Jitsi Meet instance by running docker-compose up -d with the official Jitsi Docker images on a VPS with at least 2GB RAM and a domain name"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /jitsi-meet-self-hosted-setup-guide/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---
{% raw %}

Deploy a self-hosted Jitsi Meet instance by running `docker-compose up -d` with the official Jitsi Docker images on a VPS with at least 2GB RAM and a domain name pointed to your server. You will have a working, private video conferencing server with automatic Let's Encrypt SSL in under 30 minutes. This guide covers the full setup -- Docker Compose configuration, authentication, TURN server configuration, security hardening, and horizontal scaling with additional video bridge instances.

## Why Self-Host Jitsi Meet

Public Jitsi Meet instances work adequately for casual use, but self-hosting delivers significant advantages. You control all conference metadata—meeting durations, participant counts, and usage patterns remain private. Custom branding becomes possible, including themed interfaces and personalized welcome pages. For organizations with compliance requirements, self-hosting ensures data never leaves your infrastructure.

The trade-offs merit consideration. Self-hosting requires ongoing maintenance, SSL certificate management, and infrastructure costs. For small teams, the public instances may suffice. However, developers building custom integrations or organizations prioritizing privacy will find self-hosting worthwhile.

## Prerequisites and Infrastructure

Before beginning, ensure your server meets these requirements:

- A VPS or dedicated server with at least 2GB RAM (4GB recommended for stable multi-party calls)
- Ubuntu 20.04 LTS or newer
- A domain name pointing to your server's IP address
- Docker and Docker Compose installed

Install Docker on Ubuntu:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

### Step 1: Docker Deployment

The Jitsi project maintains official Docker images that simplify deployment significantly. Create a directory for your installation:

```bash
mkdir -p ~/jitsi-meet/{config,transcripts,recordings}
cd ~/jitsi-meet
```

Create a `.env` configuration file based on the provided template:

```bash
# .env configuration
HTTP_PORT=80
HTTPS_PORT=443
TZ=America/New_York

# Generate secure random strings for these values
JICOFO_COMPONENT_SECRET=
JICOFO_AUTH_PASSWORD=
JVB_AUTH_PASSWORD=
```

Generate the required secrets:

```bash
# Generate secure passwords
openssl rand -base64 24  # Use output for JICOFO_AUTH_PASSWORD
openssl rand -base64 24  # Use output for JVB_AUTH_PASSWORD
openssl rand -base64 24  # Use output for JICOFO_COMPONENT_SECRET
```

Create your `docker-compose.yml`:

```yaml
version: '3'

services:
    # Frontend web service
    web:
        image: jitsi/web:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        ports:
            - '${HTTP_PORT}:80'
            - '${HTTPS_PORT}:443'
        volumes:
            - ./config:/config
            - ./transcripts:/transcripts
        environment:
            - CONFIG_EXTERNAL_DOMAIN
            - CONFIG_WHOOGLE_ENABLED=1
            - JICOFO_AUTH_PASSWORD
            - JVB_AUTH_PASSWORD
            - JICOFO_COMPONENT_SECRET
        depends_on:
            - prosody

    # XMPP server
    prosody:
        image: jitsi/prosody:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        volumes:
            - ./config/prosody:/config
        environment:
            - JICOFO_AUTH_PASSWORD
            - JVB_AUTH_PASSWORD
            - JICOFO_COMPONENT_SECRET

    # Video bridge
    jvb:
        image: jitsi/jvb:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        ports:
            - '10000:10000/udp'
        volumes:
            - ./config/jvb:/config
        environment:
            - JVB_AUTH_PASSWORD
            - PUBLIC_URL

    # Focus controller
    jicofo:
        image: jitsi/jicofo:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        volumes:
            - ./config/jicofo:/config
        environment:
            - JICOFO_AUTH_PASSWORD
            - JICOFO_COMPONENT_SECRET
```

Configure your domain in a `.env` file:

```bash
# .env
CONFIG_EXTERNAL_DOMAIN=meet.yourdomain.com
PUBLIC_URL=https://meet.yourdomain.com
JICOFO_AUTH_PASSWORD=your_generated_password
JVB_AUTH_PASSWORD=your_generated_password
JICOFO_COMPONENT_SECRET=your_generated_secret
RESTART_POLICY=unless-stopped
```

Start the services:

```bash
docker-compose up -d
```

After startup completes, access your instance at `https://meet.yourdomain.com`. The first visit initializes the SSL certificates through Let's Encrypt automatically.

### Step 2: Essential Configuration Options

Jitsi Meet provides extensive customization through environment variables. These settings belong in your `.env` file or the web service configuration.

**Authentication Configuration:**

By default, your deployment allows anyone to create meetings. Secure it by requiring authentication:

```bash
# Enable authentication - only logged-in users can start meetings
JICOFO_AUTH_TYPE=internal
ENABLE_AUTH=1
ENABLE_GUESTS=1  # Allow guests to join existing meetings
```

With authentication enabled, users must create accounts through the Prosody admin interface. Create a user:

```bash
docker exec -it prosody prosodyctl adduser admin@meet.yourdomain.com
```

**Custom Branding:**

Replace the default Jitsi branding with your own:

```bash
# Custom interface config
CONFIG_INTERFACE_NAME=Your Organization
CONFIG_WELCOME_PAGE_CUSTOMIZATION_ENABLED=1
```

Add your logo by mounting a custom interface config:

```bash
# In docker-compose, add to web service volumes:
- ./custom-interfaceConfig.json:/config/interfaceConfig.conf.json:ro
```

**Recording Capabilities:**

Enable local recording (note: recording requires authentication):

```bash
ENABLE_RECORDING=1
```

### Step 3: STUN and TURN Configuration

For connections through restrictive firewalls or NAT, configure TURN servers. The default Jitsi Meet deployment includes STUN, but production deployments benefit from dedicated TURN infrastructure:

```bash
# Use Google's STUN servers or deploy your own
JVB_STUN_SERVERS=stun.l.google.com:19302

# Coturn installation required for TURN
# Add to docker-compose for TURN support:
TURN_HOST=turn.yourdomain.com
TURN_PORT=443
TURNS_PORT=443
```

Deploying your own Coturn server improves connectivity for users behind corporate firewalls and symmetric NAT.

### Step 4: Security Hardening

After basic installation, implement these security measures:

**Rate Limiting:**

Jitsi Meet includes built-in rate limiting through the XMPP server. Adjust settings in `config/prosody/prosody.cfg.lua`:

```lua
limits = {
    c2s = {
        rate = "10kb/s";
        burst = "20kb";
    };
}
```

**TLS Configuration:**

The automatic Let's Encrypt certificates work well, but verify your deployment uses modern TLS settings. The included Nginx configuration provides reasonable defaults, but restrict older protocols for production:

```nginx
# In your custom Nginx configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers HIGH:!aNULL:!MD5;
ssl_prefer_server_ciphers on;
```

**Firewall Configuration:**

Configure your firewall to permit only necessary traffic:

```bash
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow 10000/udp # JVB media
sudo ufw enable
```

### Step 5: Scaling Considerations

Single-server deployments support approximately 20-30 simultaneous participants depending on available bandwidth. For larger deployments, scale horizontally by deploying additional JVB (Video Bridge) instances:

```yaml
# Add to docker-compose for additional bridges
jvb2:
    image: jitsi/jvb:latest
    restart: unless-stopped
    ports:
        - '10001:10000/udp'
    environment:
        - JVB_AUTH_PASSWORD
        - PUBLIC_URL
        - JVB_BOSH_URL_BASE=http://prosody:5280
```

Configure the load balancer to distribute WebSocket connections across all JVB instances.

## Troubleshooting Common Issues

**Audio or Video Not Connecting:**

Check that port 10000 UDP is open and not blocked by your ISP or firewall. Verify JVB logs for connection errors:

```bash
docker logs jvb
```

**Certificate Errors:**

If certificates fail to provision automatically, manually obtain them:

```bash
docker-compose stop web
sudo certbot certonly -d meet.yourdomain.com
# Copy certificates to config directory
docker-compose start web
```

**Meeting Not Starting:**

Examine Prosody logs for authentication or room creation issues:

```bash
docker logs prosody
```

## Frequently Asked Questions

**How long does it take to guide?**

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

- [How To Set Up Jitsi Meet Self Hosted Encrypted Video](/how-to-set-up-jitsi-meet-self-hosted-encrypted-video-confere/)
- [Self-Hosted Private Video Calling Setup Guide](/private-video-calling-selfhosted-guide/)
- [Privacy Setup For Psychologist Telehealth Sessions Encrypted](/privacy-setup-for-psychologist-telehealth-sessions-encrypted/)
- [How To Set Up Self Hosted Matrix Synapse Server For Private](/how-to-set-up-self-hosted-matrix-synapse-server-for-private-/)
- [Nextcloud Talk Video Calls Setup Guide](/nextcloud-talk-video-calls-setup-guide/)
- [Best Self-Hosted AI Model for JavaScript TypeScript Code](https://bestremotetools.com/best-self-hosted-ai-model-for-javascript-typescript-code-gen/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
