---
layout: default
title: "Self-Hosted Private Video Calling Setup Guide"
description: "Set up Jitsi Meet, Matrix Element, and Galene as self-hosted video calling alternatives to Zoom and Google Meet, with E2EE and no third-party data collection"
date: 2026-03-22
author: theluckystrike
permalink: /private-video-calling-selfhosted-guide/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Self-Hosted Private Video Calling Setup Guide

Zoom sends meeting metadata to Amazon, Google Meet processes video through Google's infrastructure, and Microsoft Teams is tied to a Microsoft account. Self-hosted video calling gives you control over who has access to call data, recordings, and participant information. This guide covers three options: Jitsi Meet (easiest to deploy), Matrix + Element (federated, persistent rooms), and Galene (minimal, for small groups).

---

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

## Related Reading

- [Best Secure Video Calling App 2026](/privacy-tools-guide/best-secure-video-calling-app-2026/)
- [Nextcloud Talk Video Calls Setup Guide](/privacy-tools-guide/nextcloud-talk-video-calls-setup-guide/)
- [Privacy Calendar and Contacts Sync Guide](/privacy-tools-guide/privacy-calendar-contacts-sync-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
