---
layout: default
title: "How To Set Up Jitsi Meet Self Hosted Encrypted Video Confere"
description: "A practical guide for developers and power users to deploy a self-hosted Jitsi Meet server with end-to-end encryption. Covers Docker deployment"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-set-up-jitsi-meet-self-hosted-encrypted-video-confere/
categories: [guides]
intent-checked: true
voice-checked: true
reviewed: true
score: 8
tags: [privacy-tools-guide]
---

{% raw %}

Deploying a self-hosted Jitsi Meet instance with end-to-end encryption gives you complete control over your video conferencing infrastructure. This guide walks through setting up Jitsi Meet with Docker, enabling encryption, and hardening the deployment for production use in 2026.

## Understanding Jitsi Meet Encryption

Jitsi Meet provides two layers of encryption: transport layer security (TLS) for the connection between clients and the server, and end-to-end encryption (E2EE) for the actual media streams. The distinction matters significantly for privacy-conscious deployments.

Transport encryption protects your video calls from eavesdropping on the network between participants and your server. End-to-end encryption goes further—media gets encrypted on the sending device and can only be decrypted on the receiving devices. Even server administrators cannot access the content of E2EE-protected calls.

Jitsi Meet supports E2EE using the WebRTC Insertable Streams API, which allows JavaScript to process media frames before transmission. This approach provides strong encryption without requiring external plugins.

## Server Requirements and Preparation

For a basic production deployment, you need:

- A VPS or dedicated server with at least 4GB RAM and 2 CPU cores
- A domain name with DNS properly configured
- Docker and Docker Compose installed
- At least 20GB of storage for recordings and logs

Start by ensuring your server has Docker installed:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
docker --version
```

Create a directory for your Jitsi configuration:

```bash
mkdir -p ~/jitsi/{/.jitsi-meet-cfg,/{transcripts,recordings,logs}}
cd ~/jitsi
```

## Docker Compose Configuration

Create your docker-compose.yml file with production-ready settings:

```yaml
version: '3.8'

services:
    # Frontend web interface
    web:
        image: jitsi/web:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        ports:
            - '${HTTP_PORT:-80}:80'
            - '${HTTPS_PORT:-443}:443}
        volumes:
            - ./.jitsi-meet-cfg/web:/config:Z
            - ./transcripts:/usr/share/jitsi-meet/transcripts:Z
        environment:
            - ENABLE_LETSENCRYPT=1
            - LETSENCRYPT_DOMAIN=${DOMAIN}
            - LETSENCRYPT_EMAIL=${EMAIL}
            - PUBLIC_URL=${PUBLIC_URL}
            - ENABLE_GUESTS=0
            - ENABLE_AUTH=1
            - AUTH_TYPE=internal
            - JICOFO_AUTH_USER=focus
            - JVB_AUTH_USER=jvb
            - ENABLE_RECORDING=0
            - ENABLE_END_TO_END_ENCRYPTION=1

    # Focus component handles conference coordination
    jicofo:
        image: jitsi/jicofo:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        volumes:
            - ./.jitsi-meet-cfg/jicofo:/config:Z
        environment:
            - ENABLE_AUTH=1
            - AUTH_TYPE=internal
            - JICOFO_AUTH_USER=focus
            - XMPP_SERVER=xmpp
            - JVB_BREWERY_MUC=jvbbrewry

    # Video bridge handles media routing
    jvb:
        image: jitsi/jvb:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        ports:
            - '${JVB_PORT:-10000}:10000/udp'
            - '${JVB_TCP_PORT:-4443}:4443}
        volumes:
            - ./.jitsi-meet-cfg/jvb:/config:Z
        environment:
            - JVB_AUTH_USER=jvb
            - JVB_BREWERY_MUC=jvbbrewry
            - XMPP_SERVER=xmpp
            - JVB_PORT=10000
            - JVB_TCP_HARVESTER_DISABLED=true

    # XMPP server for signaling
    xmpp:
        image: jitsi/prosody:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        volumes:
            - ./.jitsi-meet-cfg/prosody:/config:Z
        environment:
            - AUTH_TYPE=internal
            - ENABLE_GUESTS=0
            - JICOFO_AUTH_USER=focus
            - JVB_AUTH_USER=jvb

    # Ethereal LDA for message handling
    etherpad:
        image: jitsi/etherpad:latest
        restart: ${RESTART_POLICY:-unless-stopped}
        ports:
            - '9001:9001'
        volumes:
            - ./.jitsi-meet-cfg/etherpad:/data:Z
```

Create your .env file with your specific values:

```bash
# .env configuration
DOMAIN=jitsi.yourdomain.com
PUBLIC_URL=https://jitsi.yourdomain.com
EMAIL=admin@yourdomain.com
JVB_PORT=10000
JVB_TCP_PORT=4443
HTTP_PORT=80
HTTPS_PORT=443
RESTART_POLICY=unless-stopped
```

## Enabling End-to-End Encryption

Jitsi Meet's E2EE requires no additional configuration in recent versions—it activates through the UI. However, certain settings ensure optimal operation:

The `ENABLE_END_TO_END_ENCRYPTION=1` environment variable enables the encryption toggle in the interface. When enabled, participants see a shield icon in the toolbar. Clicking it activates E2EE for that participant's outgoing streams.

Note that E2EE has trade-offs. Screen sharing quality may decrease, and some features like recording become unavailable during encrypted calls. Participants using certain mobile apps may experience compatibility issues.

For organizations requiring E2EE, communicate these limitations to users before deployment.

## User Authentication Setup

Internal authentication requires users to register before hosting meetings. Configure this in the Prosody configuration:

```bash
# Create user accounts via prosodyctl
docker exec -it jitsi-xmpp prosodyctl register user yourdomain.com password
```

Host authentication prevents unauthorized users from starting meetings. Guests can still join meetings started by authenticated users, but they cannot initiate new conferences.

For tighter control, consider LDAP or OAuth integration by modifying the prosody configuration files in `.jitsi-meet-cfg/prosody/`.

## TURN Server Configuration

TURN servers enable connectivity when participants are behind restrictive firewalls or NAT. Without a TURN server, some users cannot establish peer connections.

Jitsi Meet can use coturn, an open-source TURN server. Add this to your Docker Compose:

```yaml
  # TURN server for NAT traversal
  turn:
    image: coturn/coturn:latest
    restart: ${RESTART_POLICY:-unless-stopped}
    network_mode: host
    volumes:
        - ./jitsi-meet-cfg/turn:/etc/coturn
    environment:
        - LISTEN_PORT=3478
        - EXTERNAL_IP=$(curl -s ifconfig.me)
        - TURNSERVER_ENABLED=1
```

Configure Jitsi to use the TURN server by adding to the web service environment:

```bash
TURN_SERVER=turn:yourdomain.com:3478
TURN_SECRET=your_turn_secret
```

Generate the secret using a random string generator and store it securely.

## Network and Firewall Configuration

For production deployments, configure your firewall appropriately:

```bash
# Allow necessary ports
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow 10000/udp # JVB UDP
sudo ufw allow 3478/udp  # STUN/TURN
sudo ufw allow 3478/tcp  # TURN TCP
sudo ufw enable
```

If using a cloud provider, ensure security groups permit this traffic.

## SSL Certificate Management

The configuration enables Let's Encrypt automatically on first deployment. For production environments, consider using certbot for manual certificate management:

```bash
sudo certbot certonly --webroot -w /var/www/html -d jitsi.yourdomain.com
```

Copy the certificates to your Jitsi configuration and update the web container environment to point to them instead of using automatic Let's Encrypt.

## Starting and Testing Your Deployment

Launch the stack:

```bash
docker-compose up -d
docker-compose logs -f web
```

Check that all services start without errors. Access your instance at `https://jitsi.yourdomain.com`. Create a room and test with multiple participants if possible.

To verify encryption, open browser developer tools and inspect WebRTC connections. Look for `srtp` in the media section, confirming that media flows with secure real-time protocol encryption.

## Production Hardening Recommendations

Beyond basic setup, implement these hardening measures:

1. **Regular updates**: Pull latest images monthly and redeploy
2. **Logging**: Configure log rotation to prevent disk exhaustion
3. **Rate limiting**: Prevent abuse through nginx rate limits
4. **Monitoring**: Set up health checks and alerting
5. **Backups**: Regularly back up configuration and user data


## Related Articles

- [Jitsi Meet Self Hosted Setup Guide](/privacy-tools-guide/jitsi-meet-self-hosted-setup-guide/)
- [How To Set Up Self Hosted Matrix Synapse Server For Private](/privacy-tools-guide/how-to-set-up-self-hosted-matrix-synapse-server-for-private-/)
- [Best Self-Hosted File Sync Alternatives in 2026](/privacy-tools-guide/best-self-hosted-file-sync-alternative-2026/)
- [Bitwarden Self-Hosted Setup Guide](/privacy-tools-guide/bitwarden-self-hosted-setup-guide/)
- [Bitwarden vs Vaultwarden Self-Hosted: A Technical Comparison](/privacy-tools-guide/bitwarden-vs-vaultwarden-self-hosted-comparison/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
