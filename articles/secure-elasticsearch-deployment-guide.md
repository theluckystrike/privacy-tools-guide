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

Installation

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install elasticsearch
Do NOT start yet. configure security first
```

Step 1 - Configure elasticsearch.yml

```yaml
cluster.name: production-cluster
node.name: node-01

NEVER use 0.0.0.0 in production
network.host: 127.0.0.1
http.port: 9200

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

Security (enabled by default in ES 8)
xpack.security.enabled: true
xpack.security.enrollment.enabled: false

TLS for HTTP layer
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /etc/elasticsearch/certs/http.p12

TLS for transport layer
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/transport.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/transport.p12

Audit logging
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: >
  authentication_success,authentication_failed,access_denied,connection_denied
```

Step 2 - Generate TLS Certificates

```bash
Generate CA
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca \
  --out /etc/elasticsearch/certs/elastic-ca.p12 --pass ""

Generate transport cert
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/certs/elastic-ca.p12 --ca-pass "" \
  --out /etc/elasticsearch/certs/transport.p12 --pass ""

Generate HTTP cert (interactive)
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil http

sudo chmod 640 /etc/elasticsearch/certs/*.p12
sudo chown elasticsearch:elasticsearch /etc/elasticsearch/certs/*.p12
```

Step 3 - Set Passwords for Built-In Users

```bash
sudo systemctl start elasticsearch

Auto-generate passwords for all built-in users
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
Save the output. shows passwords for elastic, kibana_system, logstash_system, etc.

Test authentication
curl -k -u elastic:YOUR_PASSWORD https://127.0.0.1:9200/
```

Step 4 - Role-Based Access Control

```bash
Read-only role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/role/read_only" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{
      "names": ["*"],
      "privileges": ["read", "view_index_metadata"]
    }]
  }'

Logstash ingestion role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/role/logstash_writer" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{
      "names": ["logs-*", "metrics-*"],
      "privileges": ["create_index", "index", "create"]
    }]
  }'

Create a user
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/user/analyst" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "SecureAnalystPass123!",
    "roles": ["read_only"],
    "full_name": "Data Analyst"
  }'
```

Step 5 - Network Isolation with UFW

```bash
sudo ufw deny 9200/tcp
sudo ufw deny 9300/tcp

Only allow from specific app servers
sudo ufw allow from 10.0.0.5 to any port 9200 proto tcp

sudo ufw reload
```

Step 6 - Monitor Audit Logs

```bash
tail -f /var/log/elasticsearch/*.audit.json | \
  jq 'select(.event.type == "authentication_failed") |
      {time: .["@timestamp"], user: .user.name, ip: .origin.address}'
```

Verify Security Configuration

```bash
Unauthenticated access should fail with 401
curl -k https://127.0.0.1:9200/

Authenticated access
curl -k -u elastic:YOUR_PASSWORD https://127.0.0.1:9200/_cluster/health
```

Multi-Node Cluster Security

Elasticsearch clusters have internal communication (transport layer) that must also be secured:

```bash
Generate transport certificates for multi-node cluster
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --name node-01 \
  --dns node-01.example.com \
  --ip 10.0.0.10 \
  --ca /etc/elasticsearch/certs/elastic-ca.p12

Repeat for each node with unique names and IPs
```

In `elasticsearch.yml` for node-01:

```yaml
cluster.name: production-cluster
node.name: node-01
node.roles: [data, master]

network.host: 10.0.0.10
discovery.seed_hosts: ["10.0.0.10", "10.0.0.11", "10.0.0.12"]
cluster.initial_master_nodes: ["node-01", "node-02", "node-03"]

TLS for transport (node-to-node)
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/node-01.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/elastic-ca.p12
```

Each node verifies every other node's certificate. An attacker cannot join the cluster without valid certificates.

Index-Level Encryption

Even with TLS, data on disk is unencrypted unless you use LUKS. Add full-disk encryption:

```bash
Install cryptsetup
sudo apt install cryptsetup

Create encrypted volume for Elasticsearch data
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup luksOpen /dev/sdb1 elasticsearch-data

Format and mount
sudo mkfs.ext4 /dev/mapper/elasticsearch-data
sudo mount /dev/mapper/elasticsearch-data /var/lib/elasticsearch

Update elasticsearch.yml
path.data: /var/lib/elasticsearch
```

With full-disk encryption, if someone steals the physical drive, data is unreadable without the encryption key.

Snapshot Security

Elasticsearch snapshots can be stored on S3, local filesystem, or other backends. Secure them:

```bash
Configure S3 repository with encryption
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_snapshot/my_s3_repo" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "my-elasticsearch-backups",
      "region": "us-east-1",
      "server_side_encryption": true,
      "storage_class": "STANDARD_IA",
      "base_path": "elasticsearch",
      "compress": true
    }
  }'

Create snapshot
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_snapshot/my_s3_repo/snapshot_$(date +%s)"
```

Always enable:
- Server-side encryption (S3 encrypts at rest)
- Compression (reduces snapshot size and bandwidth)
- Separate AWS account or IAM role for snapshot access

Preventing Common Elasticsearch Breaches

Breach Pattern 1 - Unauthenticated Access

Public ES instances have been compromised thousands of times. Always:

```yaml
xpack.security.enabled: true
network.host: 127.0.0.1  # Never 0.0.0.0
```

Breach Pattern 2 - Default Credentials

Elasticsearch comes with default passwords. Always:

```bash
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
```

This generates random passwords for built-in users.

Breach Pattern 3 - Privilege Escalation

Users with read access to one index shouldn't read all indexes:

```bash
Create minimal privilege role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/role/logs_reader" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [{
      "names": ["logs-*"],
      "privileges": ["read"]
    }]
  }'

Create user with only logs_reader role
curl -k -u elastic:YOUR_PASSWORD \
  -X PUT "https://127.0.0.1:9200/_security/user/logstash" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "StrongLogstashPassword",
    "roles": ["logs_reader"]
  }'
```

Logstash now reads only `logs-*` indices, cannot access user data or configuration.

Breach Pattern 4 - Insufficient Audit Logging

Configure detailed auditing:

```yaml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.enabled: true
xpack.security.audit.logfile.events.include:
  - authentication_success
  - authentication_failed
  - access_denied
  - index_access
  - connection_granted
  - connection_denied
```

Monitor audit logs for suspicious activity:

```bash
Failed authentication attempts
tail -f /var/log/elasticsearch/*.audit.json | \
  jq 'select(.event.type == "authentication_failed")'

Unauthorized access attempts
tail -f /var/log/elasticsearch/*.audit.json | \
  jq 'select(.event.type == "access_denied")'
```

Performance Impact of Security

Security adds overhead:

- TLS encryption: ~10-15% CPU cost
- Authentication checks: Minimal (~2%)
- Audit logging: ~5-10% disk I/O cost

On modern hardware, this is acceptable. Elasticsearch remains fast even with security enabled.

If performance becomes critical:

1. Use separate security clusters (policy enforcement) and data clusters
2. Disable audit logging on high-throughput nodes
3. Use AES-NI hardware acceleration (available on most CPUs)

Security Checklist

Before deploying Elasticsearch to production:

- [ ] X-Pack security enabled
- [ ] TLS configured for HTTP and transport
- [ ] Passwords are 16+ characters, auto-generated
- [ ] Network access restricted to application servers only
- [ ] Unauthenticated port 9200 blocked globally
- [ ] RBAC roles created for each application
- [ ] Audit logging enabled
- [ ] Snapshots encrypted in S3 or backup storage
- [ ] Data at rest encrypted (LUKS or S3 server-side encryption)
- [ ] Elasticsearch updated to latest patch version
- [ ] Firewall rules reviewed monthly

Related Reading

- [Secure Redis Deployment Without Exposure](/secure-redis-deployment-without-exposure/)
- [Setting Up Grafana Loki for Security Logs](/setting-up-grafana-loki-for-security-logs/)
- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)
- [Best Password Manager with Secure Notes: A Technical Guide](/best-password-manager-with-secure-notes/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [Best AI Tools for Container Security Scanning in Deployment](https://bestremotetools.com/best-ai-tools-for-container-security-scanning-in-deployment-/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
