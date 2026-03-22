---
layout: default
title: "Self-Hosted Private Video Calling Setup Guide"
description: "Set up Jitsi Meet, Matrix Element, and Galene as self-hosted video calling alternatives to Zoom and Google Meet, with E2EE and no third-party data collection"
date: 2026-03-22
author: theluckystrike
permalink: /private-video-calling-selfhosted-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

# Self-Hosted Private Video Calling Setup Guide

Zoom sends meeting metadata to Amazon, Google Meet processes video through Google's infrastructure, and Microsoft Teams is tied to a Microsoft account. Self-hosted video calling gives you control over who has access to call data, recordings, and participant information. This guide covers three options: Jitsi Meet (easiest to deploy), Matrix + Element (federated, persistent rooms), and Galene (minimal, for small groups).
---

## Key Takeaways

- **Galene requires almost no maintenance after setup**: the binary has no external dependencies and uses minimal server resources.
- **---
## Option 1**: Jitsi Meet

Jitsi Meet is the most widely deployed self-hosted video conferencing platform.
- **Browser-based**: no account required for participants, and supports E2EE for small meetings.
- **CPU and memory monitoring for Jitsi**: The Jitsi Video Bridge (JVB) process is the most CPU-intensive component.
- **Choose Jitsi Meet when**: You need quick meeting links you can share with external participants who have no accounts.
- **Choose Matrix + Element when**: Your team needs persistent rooms, asynchronous chat alongside video, and the ability to federate with other Matrix servers.

## Option 1: Jitsi Meet

Jitsi Meet is the most widely deployed self-hosted video conferencing platform. Browser-based, no account required for participants, and supports E2EE for small meetings.

### Server Requirements

- 2+ CPU cores, 4GB RAM (for up to 50 concurrent users)
- Ubuntu 22.04 LTS recommended
- Domain with DNS pointing to the server

### Install Jitsi Meet

```bash
# Add Jitsi repository
curl https://download.jitsi.org/jitsi-key.gpg.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/jitsi-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] \
  https://download.jitsi.org stable/" | \
  sudo tee /etc/apt/sources.list.d/jitsi-stable.list

sudo apt update

# Install (this installs Jitsi + Nginx + Let's Encrypt)
sudo apt install jitsi-meet

# When prompted:
# - Enter your domain: meet.yourdomain.com
# - Choose SSL certificate: Let's Encrypt (automated)
# - Enter email for Let's Encrypt

# That's it — Jitsi is running at https://meet.yourdomain.com
```

### Harden Jitsi (Require Authentication)

By default, anyone can create rooms. Restrict room creation to authenticated users:

```bash
# Edit Prosody authentication config
sudo nano /etc/prosody/conf.d/meet.yourdomain.com.cfg.lua
```

```lua
-- Change authentication from "anonymous" to "internal_hashed"
VirtualHost "meet.yourdomain.com"
    authentication = "internal_hashed"

-- Keep guest access for participants (they can JOIN but not create)
VirtualHost "guest.meet.yourdomain.com"
    authentication = "anonymous"
    c2s_require_encryption = false
```

```bash
# Create user accounts for hosts
sudo prosodyctl register alice meet.yourdomain.com SecurePass123
sudo prosodyctl register bob meet.yourdomain.com AnotherPass456

# Edit Jicofo config to require auth for room creation
sudo nano /etc/jitsi/jicofo/jicofo.conf
```

```conf
jicofo {
  authentication {
    enabled = true
    type = XMPP
    login-url = meet.yourdomain.com
  }
}
```

```bash
sudo systemctl restart prosody jicofo jitsi-videobridge2
```

### Enable End-to-End Encryption

Jitsi supports E2EE for meetings with fewer than ~15 participants (requires browser's Insertable Streams API):

```bash
# /etc/jitsi/meet/meet.yourdomain.com-config.js

config.e2eping = { enabled: false };
config.e2ee = { labels: { } };

# In the interface config:
# interfaceConfig.DISABLE_FOCUS_INDICATOR = true;
```

Participants enable E2EE via the security icon (shield) in the bottom toolbar. All participants must enable it.

---

## Option 2: Matrix + Element

Matrix is a federated, open protocol for real-time communication including video calls. Element is the primary client. Self-hosting Synapse (Matrix homeserver) gives you persistent rooms, team communication, and video via Jitsi integration or native WebRTC.

### Install Synapse

```bash
# Add Matrix repository
sudo apt install -y lsb-release wget apt-transport-https
wget -qO /usr/share/keyrings/matrix-org-archive-keyring.gpg \
  https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] \
  https://packages.matrix.org/debian/ $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/matrix-org.list

sudo apt update && sudo apt install matrix-synapse-py3

# Configure
sudo nano /etc/matrix-synapse/homeserver.yaml
```

```yaml
# /etc/matrix-synapse/homeserver.yaml (key settings)
server_name: "yourdomain.com"

database:
  name: psycopg2
  args:
    user: synapse
    password: strong_db_password
    database: synapse
    host: localhost

# Disable public registration (invite-only)
enable_registration: false
enable_registration_without_verification: false

# Rate limiting
rc_messages_per_second: 0.2
rc_message_burst_count: 10

# Logging
log_config: "/etc/matrix-synapse/log.yaml"
```

```bash
# Create PostgreSQL database for Synapse
sudo -u postgres psql << 'SQL'
CREATE DATABASE synapse ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0;
CREATE USER synapse WITH PASSWORD 'strong_db_password';
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse;
SQL

sudo systemctl enable --now matrix-synapse

# Create admin user
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml \
  -u admin -p SecureAdminPassword -a http://localhost:8008
```

### Set Up Nginx for Matrix

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Matrix federation
    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 50M;
    }
}

# Federation port (Matrix server-to-server)
server {
    listen 8448 ssl;
    server_name yourdomain.com;
    location / {
        proxy_pass http://127.0.0.1:8008;
    }
}
```

### Video Calls in Matrix

Element uses WebRTC for 1:1 calls (peer-to-peer, no server involvement for the media). For group calls, integrate Jitsi:

```yaml
# In homeserver.yaml:
# Point Jitsi to your self-hosted Jitsi Meet server
jitsi:
  preferred_jitsi_domain: meet.yourdomain.com
```

In Element: **Settings → Preferences → Jitsi → Use your own server**
Enter: `https://meet.yourdomain.com`

---

## Option 3: Galene (Minimal, for Small Groups)

Galene is a small, dependency-free video conferencing server written in Go. No WebRTC SFU complexity — runs as a single binary with minimal configuration. Suitable for small team calls (up to ~20 participants).

```bash
# Download Galene
wget https://github.com/jech/galene/releases/latest/download/galene-linux-amd64.tar.gz
tar xzf galene-linux-amd64.tar.gz
cd galene

# Generate TLS certificate (or use existing)
openssl req -newkey rsa:4096 -x509 -sha256 -days 3650 \
  -nodes -out data/cert.pem -keyout data/key.pem \
  -subj "/CN=your.domain.com"

# Create a group (room)
mkdir -p groups
cat > groups/team.json << 'EOF'
{
    "op": [{"password": "OperatorPassword123"}],
    "presenter": [{"password": "PresenterPass"}],
    "other": [{"password": "JoinPassword"}]
}
EOF

# Start Galene
./galene -data data -groups groups
# Default: listens on port 8443

# Access: https://your.server.ip:8443/group/team
# Operator/presenter/viewer have different permissions
```

---

## Firewall Rules for Video Calling

Video calling requires additional ports:

```bash
# Jitsi Meet required ports:
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 4443/tcp   # JVB (Jitsi Video Bridge) fallback
sudo ufw allow 10000/udp  # JVB media traffic (primary)

# Matrix:
sudo ufw allow 8448/tcp   # Federation

# Galene:
sudo ufw allow 8443/tcp   # Web
sudo ufw allow 49152:65535/udp  # WebRTC media (STUN/TURN)
```

---

## TURN Server (Required Behind NAT)

Without a TURN server, participants behind strict NAT can't connect (corporate firewalls, some ISPs):

```bash
# Install coturn
sudo apt install coturn

# /etc/turnserver.conf
listening-port=3478
tls-listening-port=5349
realm=turn.yourdomain.com
server-name=turn.yourdomain.com
cert=/etc/letsencrypt/live/yourdomain.com/fullchain.pem
pkey=/etc/letsencrypt/live/yourdomain.com/privkey.pem
lt-cred-mech
user=turnuser:TurnPassword
log-file=/var/log/turnserver.log
no-stdout-log

# Start coturn
sudo systemctl enable --now coturn
```

```bash
# Configure Jitsi to use your TURN server
# /etc/jitsi/meet/meet.yourdomain.com-config.js:
config.p2p.stunServers = [
    { urls: "turn:turn.yourdomain.com:443?transport=tcp",
      username: "turnuser",
      credential: "TurnPassword" }
];
```

---

## Monitoring Server Health and Call Quality

After deploying a self-hosted video calling server, ongoing monitoring ensures call quality stays acceptable and the server doesn't silently degrade. Video traffic is resource-intensive, and a server that looked adequate at setup may struggle under concurrent meetings.

**CPU and memory monitoring for Jitsi**: The Jitsi Video Bridge (JVB) process is the most CPU-intensive component. Set up a simple monitoring script that alerts you when resources are constrained:

```bash
#!/bin/bash
# /usr/local/bin/jitsi-health.sh — run via cron every 5 minutes

JVB_CPU=$(ps aux | awk '/jitsi-videobridge/ && !/awk/ {print $3}')
JVB_MEM=$(ps aux | awk '/jitsi-videobridge/ && !/awk/ {print $4}')
LOAD=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | tr -d ' ')

LOG="/var/log/jitsi-health.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "$TIMESTAMP CPU=${JVB_CPU}% MEM=${JVB_MEM}% LOAD=${LOAD}" >> "$LOG"

# Alert if CPU exceeds 80%
if [ "$(echo "$JVB_CPU > 80" | bc -l)" -eq 1 ]; then
    curl -s -d "Jitsi JVB CPU at ${JVB_CPU}% — check active meetings" \
        https://ntfy.sh/your-jitsi-alerts
fi
```

**Call quality testing**: Use `mtr` to check the path quality to your server from clients' locations before hosting important meetings:

```bash
# Test UDP packet loss to your Jitsi server (port 10000 is JVB media)
sudo mtr --udp --port 10000 meet.yourdomain.com --report --report-cycles 20
```

More than 1-2% packet loss consistently will cause visible video artifacts. If you see packet loss, check whether your VPS provider has network issues, or whether a TURN relay is needed for specific clients.

**Log rotation**: Jitsi produces significant log output. Without rotation, logs fill disk over weeks:

```bash
# /etc/logrotate.d/jitsi
/var/log/jitsi/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
        systemctl reload jitsi-videobridge2 >/dev/null 2>&1 || true
    endscript
}
```

## Choosing Between Jitsi, Matrix, and Galene

Each option in this guide serves different use cases. Choosing the wrong tool for your situation creates maintenance overhead without proportional privacy benefit.

**Choose Jitsi Meet when**: You need quick meeting links you can share with external participants who have no accounts. Jitsi is the closest drop-in replacement for Zoom—send an URL, participants join from the browser, meetings end and leave no persistent state.

**Choose Matrix + Element when**: Your team needs persistent rooms, asynchronous chat alongside video, and the ability to federate with other Matrix servers. Matrix has significantly higher setup complexity than Jitsi, but the result is a complete communication platform rather than just a video solution. If your team currently uses Slack and Zoom together, Matrix + Element can replace both.

**Choose Galene when**: You need a minimal, low-overhead solution for small trusted groups (your immediate team, family, or close collaborators). Galene requires almost no maintenance after setup—the binary has no external dependencies and uses minimal server resources. It is not suitable for meetings with external participants unfamiliar with the URL format.

Resource requirements comparison:

| Platform | Min RAM | CPU | Concurrent Users | Maintenance Load |
|----------|---------|-----|------------------|-----------------|
| Jitsi Meet | 4GB | 2 cores | 50+ | Medium |
| Matrix Synapse | 2GB | 1 core | Team-sized | High |
| Galene | 512MB | 1 core | ~20 | Low |

For most privacy-focused individuals or small teams, Jitsi on a $20/month VPS handles needs well. Teams with ongoing communication requirements that currently pay for Slack will find Matrix's total cost of ownership competitive after the initial setup investment.

## Related Reading

- [Best Secure Video Calling App 2026](/privacy-tools-guide/best-secure-video-calling-app-2026/)
- [Nextcloud Talk Video Calls Setup Guide](/privacy-tools-guide/nextcloud-talk-video-calls-setup-guide/)
- [Privacy Calendar and Contacts Sync Guide](/privacy-tools-guide/privacy-calendar-contacts-sync-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

