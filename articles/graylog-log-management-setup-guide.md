---
layout: default
title: "How to Set Up Graylog for Log Management"
description: "Deploy Graylog for centralized log aggregation, configure inputs and streams, build dashboards, and set up alerts for security monitoring"
date: 2026-03-22
author: theluckystrike
permalink: /graylog-log-management-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up Graylog for Log Management

Graylog is an open-source log management platform built on top of Elasticsearch (or OpenSearch) and MongoDB. It centralizes logs from servers, network devices, applications, and containers, making it possible to search, alert, and visualize events across your entire infrastructure. This guide sets up a working single-node Graylog instance.

## Prerequisites

- Ubuntu 22.04 server with at least 4 GB RAM (8 GB recommended)
- Root or sudo access
- Java 17
- At least 20 GB disk for log storage (more for production)

## Architecture Overview

Graylog requires three components:

- **MongoDB** — stores Graylog's metadata and configuration
- **OpenSearch** — stores and indexes the actual log data
- **Graylog** — the application server, REST API, and web interface

## Step 1 — Install Java 17

```bash
sudo apt update && sudo apt install -y openjdk-17-jre-headless
java -version
# openjdk version "17.x.x"
```

## Step 2 — Install MongoDB

```bash
# Import MongoDB GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-6.0.gpg

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

sudo apt update && sudo apt install -y mongodb-org
sudo systemctl enable --now mongod

# Verify
mongosh --eval "db.runCommand({ connectionStatus: 1 })"
```

## Step 3 — Install OpenSearch

```bash
# Install OpenSearch (Elasticsearch-compatible)
curl -fsSL https://artifacts.opensearch.org/publickeys/opensearch.pgp | \
  sudo gpg --dearmor -o /usr/share/keyrings/opensearch.gpg

echo "deb [signed-by=/usr/share/keyrings/opensearch.gpg] \
  https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/opensearch.list

sudo apt update && sudo apt install -y opensearch

# Configure OpenSearch for Graylog (single node, disable security for internal use)
sudo tee /etc/opensearch/opensearch.yml > /dev/null << 'EOF'
cluster.name: graylog
node.name: node-1
discovery.type: single-node
network.host: 127.0.0.1
plugins.security.disabled: true
action.auto_create_index: false
EOF

# Set JVM heap (half your RAM)
sudo sed -i 's/-Xms1g/-Xms2g/' /etc/opensearch/jvm.options
sudo sed -i 's/-Xmx1g/-Xmx2g/' /etc/opensearch/jvm.options

sudo systemctl enable --now opensearch

# Verify
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
```

## Step 4 — Install Graylog

```bash
# Download and install Graylog repository
wget https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.deb
sudo dpkg -i graylog-6.0-repository_latest.deb
sudo apt update && sudo apt install -y graylog-server
```

## Step 5 — Configure Graylog

Generate required secrets:

```bash
# Password secret (random 96-char string)
pwgen -N 1 -s 96
# Example output: copy this as your password_secret

# SHA2 hash of your admin password
echo -n "YourAdminPassword" | sha256sum | cut -d' ' -f1
```

Edit `/etc/graylog/server/server.conf`:

```ini
# Paste the pwgen output here
password_secret = PASTE_96_CHAR_SECRET_HERE

# Paste the sha256sum output here
root_password_sha2 = PASTE_SHA256_HASH_HERE

# Web interface binding
http_bind_address = 0.0.0.0:9000

# OpenSearch connection
elasticsearch_hosts = http://127.0.0.1:9200

# MongoDB connection
mongodb_uri = mongodb://localhost/graylog

# Root user
root_username = admin
root_email = admin@localhost
root_timezone = UTC
```

## Step 6 — Start Graylog

```bash
sudo systemctl enable --now graylog-server
sudo systemctl status graylog-server

# Watch logs during startup (takes 1-2 minutes)
sudo journalctl -u graylog-server -f
```

Access the UI at `http://YOUR_SERVER_IP:9000`. Log in with `admin` and the password you hashed above.

## Step 7 — Configure Log Inputs

In the Graylog UI, navigate to **System > Inputs**. Launch the inputs your infrastructure needs:

**Syslog UDP** — for Linux servers and network equipment:
1. Click "Launch new input"
2. Select "Syslog UDP"
3. Port: 5140 (use >1024 to avoid root requirement)
4. Bind address: 0.0.0.0
5. Click Save

**GELF UDP** — for Docker containers and applications:
1. Select "GELF UDP"
2. Port: 12201
3. Click Save

Configure your Linux servers to send logs:

```bash
# /etc/rsyslog.d/99-graylog.conf
*.* @GRAYLOG_IP:5140;RSYSLOG_SyslogProtocol23Format

# Restart rsyslog
sudo systemctl restart rsyslog
```

For Docker containers, add the GELF log driver:

```bash
docker run --log-driver=gelf \
  --log-opt gelf-address=udp://GRAYLOG_IP:12201 \
  --log-opt tag="myapp" \
  myimage:latest
```

## Step 8 — Create Streams

Streams route messages to specific pipelines and dashboards. Create a "Security Events" stream:

1. Navigate to **Streams > Create Stream**
2. Name: "Security Events"
3. Add a rule: Field `facility` matches `auth` OR `authpriv`
4. Enable stream

## Step 9 — Set Up Alerts

Navigate to **Alerts > Event Definitions > Create Event Definition**.

Example: alert on repeated SSH failures:

```
Name: SSH Brute Force Attempt
Priority: High
Condition Type: Aggregation
Query: message:"Failed password" AND facility:auth
Group by: source
Condition: count() >= 5 in last 1 minute
```

Wire this to a Notification (Email, Slack webhook, PagerDuty) under **Alerts > Notifications**.

## Step 10 — Firewall Graylog

```bash
# Allow web UI from LAN only
sudo ufw allow from 192.168.1.0/24 to any port 9000
sudo ufw deny 9000

# Allow syslog from all internal hosts
sudo ufw allow from 192.168.1.0/24 to any port 5140/udp

# Allow GELF
sudo ufw allow from 192.168.1.0/24 to any port 12201/udp

sudo ufw reload
```

## Step 11 — Content Packs

Content packs are pre-configured collections of inputs, streams, dashboards, and alerts. Install community packs to get useful dashboards immediately:

In the Graylog UI: **System → Content Packs → Browse**

Useful packs available for download from the Graylog Marketplace:
- **Linux Syslog** — Streams for auth.log, SSH events, sudo usage, and system errors
- **Nginx Access Logs** — Traffic dashboards, 4xx/5xx tracking, top URIs
- **Apache Combined** — Request rate, error rate, bandwidth dashboards
- **Docker Metrics** — Container event streams

For custom content, export your configuration as a content pack to replicate it across environments:

```bash
# Export content pack via Graylog API
curl -u admin:yourpassword \
  "https://graylog.yourcompany.com/api/system/content_packs" \
  -H "X-Requested-By: cli" \
  | python3 -m json.tool > graylog-content-pack.json
```

---

## Step 12 — Long-Term Log Archiving

Graylog Enterprise supports archiving to S3. For the open-source version, schedule periodic exports to compressed files:

```bash
#!/bin/bash
# /usr/local/bin/archive-graylog-logs.sh
# Exports yesterday's logs via Graylog API and compresses to S3-compatible storage

DATE=$(date -d "yesterday" +%Y-%m-%d)
GRAYLOG_URL="http://localhost:9000"
CREDS="admin:yourpassword"
OUTPUT_DIR="/var/archive/graylog"

mkdir -p "$OUTPUT_DIR"

# Export via Graylog search API (limited to 10,000 messages per call — paginate for large volumes)
curl -s -u "$CREDS" \
  -H "Accept: text/csv" \
  "${GRAYLOG_URL}/api/views/search/messages?q=*&timerange.type=absolute&timerange.from=${DATE}T00:00:00Z&timerange.to=${DATE}T23:59:59Z&fields=timestamp,source,message,level&limit=10000" \
  | gzip > "${OUTPUT_DIR}/graylog-${DATE}.csv.gz"

# Sync to object storage (requires rclone configured)
rclone copy "${OUTPUT_DIR}/graylog-${DATE}.csv.gz" backblaze:logs/graylog/

# Retain locally for 30 days
find "$OUTPUT_DIR" -name "*.csv.gz" -mtime +30 -delete

echo "Archive complete for $DATE"
```

Add to cron:
```bash
echo "5 2 * * * root /usr/local/bin/archive-graylog-logs.sh >> /var/log/graylog-archive.log 2>&1" | \
  sudo tee /etc/cron.d/graylog-archive
```

---

## Related Reading

- [How to Set Up ntopng for Network Analytics](/privacy-tools-guide/ntopng-network-analytics-setup-guide/)
- [How to Set Up Falco for Container Runtime Security](/privacy-tools-guide/falco-container-runtime-security-setup/)
- [How to Use strace for Security Analysis](/privacy-tools-guide/strace-security-analysis-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
