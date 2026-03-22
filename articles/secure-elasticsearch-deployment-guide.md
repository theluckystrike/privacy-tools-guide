---
layout: default
title: "Secure Elasticsearch Deployment Guide"
description: "Harden Elasticsearch with TLS, X-Pack security, role-based access control, and network isolation to prevent data exposure in production"
date: 2026-03-22
author: theluckystrike
permalink: /secure-elasticsearch-deployment-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Elasticsearch has been the source of more large-scale data breaches than almost any other technology. The default configuration has no authentication and listens on all interfaces. This guide covers the full security stack: network binding, TLS, authentication, RBAC, and audit logging.

## Installation

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install elasticsearch
# Do NOT start yet — configure security first
```

## Step 1: Configure elasticsearch.yml

```yaml
cluster.name: production-cluster
node.name: node-01

# NEVER use 0.0.0.0 in production
network.host: 127.0.0.1
http.port: 9200

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Security (enabled by default in ES 8)
xpack.security.enabled: true
xpack.security.enrollment.enabled: false

# TLS for HTTP layer
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /etc/elasticsearch/certs/http.p12

# TLS for transport layer
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/transport.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/transport.p12

# Audit logging
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: >
  authentication_success,authentication_failed,access_denied,connection_denied
```

## Step 2: Generate TLS Certificates

```bash
# Generate CA
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca \
  --out /etc/elasticsearch/certs/elastic-ca.p12 --pass ""

# Generate transport cert
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/certs/elastic-ca.p12 --ca-pass "" \
  --out /etc/elasticsearch/certs/transport.p12 --pass ""

# Generate HTTP cert (interactive)
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil http

sudo chmod 640 /etc/elasticsearch/certs/*.p12
sudo chown elasticsearch:elasticsearch /etc/elasticsearch/certs/*.p12
```

## Step 3: Set Passwords for Built-In Users

```bash
sudo systemctl start elasticsearch

# Auto-generate passwords for all built-in users
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
# Save the output — shows passwords for elastic, kibana_system, logstash_system, etc.

# Test authentication
curl -k -u elastic:YOUR_PASSWORD https://127.0.0.1:9200/
```

## Step 4: Role-Based Access Control

```bash
# Read-only role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/role/read_only" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{
      "names": ["*"],
      "privileges": ["read", "view_index_metadata"]
    }]
  }'

# Logstash ingestion role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/role/logstash_writer" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{
      "names": ["logs-*", "metrics-*"],
      "privileges": ["create_index", "index", "create"]
    }]
  }'

# Create a user
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/user/analyst" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "SecureAnalystPass123!",
    "roles": ["read_only"],
    "full_name": "Data Analyst"
  }'
```

## Step 5: Network Isolation with UFW

```bash
sudo ufw deny 9200/tcp
sudo ufw deny 9300/tcp

# Only allow from specific app servers
sudo ufw allow from 10.0.0.5 to any port 9200 proto tcp

sudo ufw reload
```

## Step 6: Monitor Audit Logs

```bash
tail -f /var/log/elasticsearch/*.audit.json | \
  jq 'select(.event.type == "authentication_failed") |
      {time: .["@timestamp"], user: .user.name, ip: .origin.address}'
```

## Verify Security Configuration

```bash
# Unauthenticated access should fail with 401
curl -k https://127.0.0.1:9200/

# Authenticated access
curl -k -u elastic:YOUR_PASSWORD https://127.0.0.1:9200/_cluster/health
```

## Related Reading

- [Secure Redis Deployment Without Exposure](/privacy-tools-guide/secure-redis-deployment-without-exposure/)
- [Setting Up Grafana Loki for Security Logs](/privacy-tools-guide/setting-up-grafana-loki-for-security-logs/)
- [How to Configure UFW Firewall on Ubuntu](/privacy-tools-guide/how-to-configure-ufw-firewall-on-ubuntu/)
- [Best Password Manager with Secure Notes: A Technical Guide](/privacy-tools-guide/best-password-manager-with-secure-notes/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
