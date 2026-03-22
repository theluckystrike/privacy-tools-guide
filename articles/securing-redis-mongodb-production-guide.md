---
layout: default
title: "Securing Redis and MongoDB for Production"
description: "Harden Redis and MongoDB for production with authentication, TLS encryption, network binding, access control lists, and audit logging to prevent data exposure"
date: 2026-03-22
author: theluckystrike
permalink: /securing-redis-mongodb-production-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Securing Redis and MongoDB for Production

Redis and MongoDB have both been responsible for massive data breaches — not because of software vulnerabilities, but because instances were deployed with default settings that expose data to the internet with no authentication. Redis alone had hundreds of thousands of publicly accessible instances in 2023. This guide covers authentication, network binding, TLS, and access control for both.
---

# Read-only monitoring user
user readonly on >MonitorPass!
- **Mastering advanced features takes**: 1-2 weeks of regular use.

## Table of Contents

- [The Default-Open Problem](#the-default-open-problem)
- [Redis Security](#redis-security)
- [MongoDB Security](#mongodb-security)
- [Firewall Rules for Both Services](#firewall-rules-for-both-services)
- [Verify Your Hardening](#verify-your-hardening)
- [Related Reading](#related-reading)

## The Default-Open Problem

**Redis defaults**:
- Listens on `0.0.0.0:6379` — accessible from any IP
- No authentication required
- Commands allow reading/writing all keys
- The `CONFIG SET` command can write files anywhere on the filesystem

**MongoDB defaults** (older versions):
- Listens on `0.0.0.0:27017` with no auth
- Any connection can read/write any database

If you deploy either without configuration changes on a server with a public IP, attackers will find and exploit it within hours (automated scanners run continuously).

---

## Redis Security

### Step 1: Bind to Localhost or Specific IP

```bash
# /etc/redis/redis.conf (Debian/Ubuntu) or /etc/redis.conf (RHEL)

# Only accept connections from localhost
bind 127.0.0.1

# Or bind to a specific internal IP only
# bind 192.168.1.10

# Enable protected mode (denies external connections if no auth configured)
protected-mode yes
```

```bash
# Verify binding
ss -tlnp | grep 6379
# Should show: 127.0.0.1:6379, NOT 0.0.0.0:6379

sudo systemctl restart redis
```

### Step 2: Enable Authentication

```bash
# Generate a strong password
openssl rand -base64 32
# Copy the output

# /etc/redis/redis.conf:
requirepass "YOUR_STRONG_PASSWORD_HERE"

# ACL-based authentication (Redis 6+, preferred)
# Disable default user, create named users
aclfile /etc/redis/users.acl
```

```bash
# /etc/redis/users.acl
# Syntax: user <name> [flags] [passwords] [commands] [keys]

# Disable the default user (catches unauthenticated access)
user default off nopass nocommands nokeys

# Application user: full access to app-specific keys only
user appuser on >StrongPassword123! ~app:* +@all

# Read-only monitoring user
user readonly on >MonitorPass! ~* +@read -@dangerous

# Admin user with all permissions (for management only)
user admin on >AdminPassword456! ~* +@all
```

```bash
# Reload ACL
redis-cli -a AdminPassword456! ACL LOAD

# Test authentication
redis-cli -a StrongPassword123! PING
# Expected: PONG

redis-cli PING
# Expected: NOAUTH Authentication required
```

### Step 3: Disable Dangerous Commands

```bash
# /etc/redis/redis.conf

# Rename dangerous commands to empty string (disables them)
# or rename to a secret string (hides them)
rename-command CONFIG ""
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command KEYS ""     # Use SCAN instead — KEYS blocks server
rename-command DEBUG ""
rename-command EVAL ""     # Disable if Lua scripting not needed
```

**Why `CONFIG` is dangerous**: An attacker with Redis access can use `CONFIG SET dir /home/user/.ssh` + `CONFIG SET dbfilename authorized_keys` + `SAVE` to write their SSH key to the server, gaining shell access.

### Step 4: Enable TLS (Redis 6+)

```bash
# Generate TLS certificates
openssl req -newkey rsa:4096 -x509 -sha256 -days 3650 \
  -nodes -out /etc/redis/tls/redis.crt \
  -keyout /etc/redis/tls/redis.key \
  -subj "/CN=redis-server"

# /etc/redis/redis.conf:
tls-port 6380
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients yes   # Require client certificates

# Keep port 6379 for localhost only (no TLS needed for same-host connections)
port 6379
bind 127.0.0.1
```

```bash
# Connect via TLS
redis-cli --tls \
  --cert /path/to/client.crt \
  --key /path/to/client.key \
  --cacert /etc/redis/tls/ca.crt \
  -p 6380 \
  -a StrongPassword123! PING
```

### Step 5: Persistence and File Permissions

```bash
# /etc/redis/redis.conf
# Run Redis as non-root user
# (typically handled by systemd unit — verify)

# Restrict RDB/AOF file locations
dir /var/lib/redis
dbfilename dump.rdb

# Set file permissions
chmod 750 /var/lib/redis
chown redis:redis /var/lib/redis

# Verify Redis runs as redis user, not root
ps aux | grep redis
# Should show: redis   ... redis-server
```

---

## MongoDB Security

### Step 1: Enable Authentication

```yaml
# /etc/mongod.conf

security:
  authorization: enabled

net:
  bindIp: 127.0.0.1  # localhost only
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/tls/mongod.pem
    CAFile: /etc/mongodb/tls/ca.pem
```

```bash
# Start MongoDB initially WITHOUT auth to create the admin user
# Then enable auth

# Connect without auth:
mongosh

# Create admin user
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),  // prompts securely
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"]
})
# Set a strong password when prompted

exit
```

```bash
# Enable auth in /etc/mongod.conf and restart
sudo systemctl restart mongod

# Test auth
mongosh --username admin --password --authenticationDatabase admin
# Should prompt for password and connect

mongosh --username wronguser --password badpass
# Should fail: Authentication failed
```

### Step 2: Create Role-Based Users

```javascript
// Connect as admin: mongosh -u admin -p --authenticationDatabase admin

use myapp_db

// Application user — only accesses its own database
db.createUser({
  user: "appuser",
  pwd: "StrongAppPassword!",
  roles: [
    { role: "readWrite", db: "myapp_db" }
  ]
})

// Read-only analytics user
db.createUser({
  user: "analytics",
  pwd: "AnalyticsReadOnly!",
  roles: [
    { role: "read", db: "myapp_db" }
  ]
})

// Backup user (needs backup role, not read/write)
use admin
db.createUser({
  user: "backup",
  pwd: "BackupPassword!",
  roles: ["backup"]
})
```

### Step 3: Enable TLS for MongoDB

```bash
# Generate MongoDB certificate (combined key + cert PEM)
openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/mongodb/tls/mongod.key \
  -x509 -days 3650 \
  -out /etc/mongodb/tls/mongod.crt \
  -subj "/CN=mongodb-server"

# Combine into PEM for MongoDB
cat /etc/mongodb/tls/mongod.key /etc/mongodb/tls/mongod.crt > /etc/mongodb/tls/mongod.pem
chmod 600 /etc/mongodb/tls/mongod.pem
chown mongodb:mongodb /etc/mongodb/tls/mongod.pem
```

```yaml
# /etc/mongod.conf
net:
  bindIp: 127.0.0.1,192.168.1.10   # localhost + internal IP
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/tls/mongod.pem
```

```bash
# Connect via TLS
mongosh --tls \
  --tlsCAFile /etc/mongodb/tls/ca.pem \
  --username admin --password \
  --authenticationDatabase admin \
  "mongodb://localhost:27017/"
```

### Step 4: Enable Audit Logging (Enterprise) or mongocryptd

For Community Edition, enable operation logging:

```yaml
# /etc/mongod.conf
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

operationProfiling:
  mode: slowOp       # Log slow operations (> 100ms)
  slowOpThresholdMs: 100
```

```javascript
// Check profiler status
use myapp_db
db.getProfilingStatus()

// Enable profiling for all operations (use carefully — performance impact)
db.setProfilingLevel(1, { slowms: 100 })

// Query the system.profile collection to see recent operations
db.system.profile.find().sort({ts:-1}).limit(5).pretty()
```

---

## Firewall Rules for Both Services

```bash
# Using ufw (Ubuntu)
sudo ufw deny 6379   # Block Redis from all external access
sudo ufw deny 27017  # Block MongoDB from external access

# Allow only from specific app server IPs:
sudo ufw allow from 192.168.1.20 to any port 6379
sudo ufw allow from 192.168.1.20 to any port 27017

# Using nftables directly:
sudo nft add rule inet filter input tcp dport 6379 ip saddr != 127.0.0.1 drop
sudo nft add rule inet filter input tcp dport 27017 ip saddr != { 127.0.0.1, 192.168.1.0/24 } drop
```

---

## Verify Your Hardening

```bash
# Test Redis from external IP (should fail)
redis-cli -h your.server.ip PING
# Expected: Could not connect / Connection refused

# Test MongoDB from external (should fail)
mongosh "mongodb://your.server.ip:27017"
# Expected: Connection refused or timeout

# Scan your own server for exposed ports
nmap -p 6379,27017 your.public.ip
# Expected: filtered or closed for both

# Check Redis ACL is working
redis-cli -h 127.0.0.1 KEYS "*"
# Expected: NOAUTH or permission denied (without credentials)
```

---

## Related Articles

- [Secure Redis Deployment Without Exposure](/privacy-tools-guide/secure-redis-deployment-without-exposure/)
- [How To Anonymize User Data In Production Database](/privacy-tools-guide/how-to-anonymize-user-data-in-production-database-for-privac/)
- [GDPR Compliant User Authentication](/privacy-tools-guide/gdpr-compliant-user-authentication-design/)
- [How To Build Privacy Dashboard For Customers To Manage](/privacy-tools-guide/how-to-build-privacy-dashboard-for-customers-to-manage-their/)
- [How to Secure PostgreSQL for Production](/privacy-tools-guide/secure-postgresql-production-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does Redis offer a free tier?**

Most major tools offer some form of free tier or trial period. Check Redis's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.
{% endraw %}
