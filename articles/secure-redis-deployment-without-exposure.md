---
layout: default
title: "Secure Redis Deployment Without Exposure"
description: "Harden Redis by binding to localhost, enabling authentication, using TLS, and configuring ACLs to prevent unauthorized access and data theft"
date: 2026-03-22
author: theluckystrike
permalink: /secure-redis-deployment-without-exposure/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Redis has no authentication by default and listens on all interfaces. Thousands of Redis instances are exposed to the internet with no password. attackers scan for them, dump their contents, and use them as launching pads for further attacks. This guide covers every layer of Redis hardening from network binding to ACLs.

Step 1: Bind to Localhost Only

The most critical setting. Open `/etc/redis/redis.conf`:

```bash
sudo nano /etc/redis/redis.conf
```

Find and set:
```conf
Bind only to localhost and Unix socket
bind 127.0.0.1 ::1

Or bind to a specific internal IP if app servers need access
bind 127.0.0.1 10.0.0.5

Disable listening on external interfaces completely
protected-mode yes
```

Never set `bind 0.0.0.0` on a production Redis instance. If application servers on other machines need Redis access, use a private network interface or SSH tunnel. not public exposure.

Step 2: Enable Authentication

Simple Password (Redis 6+ Legacy)

```conf
In redis.conf
requirepass "use-a-very-long-random-password-here"
```

Generate a strong password:
```bash
openssl rand -base64 48
Output: something like Xk7mP2nQ8vR3tL5sJ9wH1dF4...
```

ACL-Based Authentication (Redis 6+ Recommended)

Redis 6 introduced ACLs (Access Control Lists) for fine-grained per-user permissions. Create an ACL file:

```bash
sudo nano /etc/redis/users.acl
```

```
Disable the default user (critical. default user has full access)
user default off nopass nocommands

Read-only user for monitoring
user monitor on >monitorpassword ~* &* +INFO +MONITOR +KEYS +GET +HGET +LRANGE

Application user: full key access but cannot modify server config
user appuser on >strongapppassword ~* &* +@read +@write +@string +@hash +@list +@set +@sortedset -CONFIG -DEBUG -SHUTDOWN -REPLICAOF -SLAVEOF

Admin user: full access (use only for admin tasks)
user admin on >adminpassword allkeys allchannels allcommands
```

Reference the ACL file in redis.conf:
```conf
aclfile /etc/redis/users.acl
```

Connect with a specific user:
```bash
redis-cli -u redis://appuser:strongapppassword@127.0.0.1:6379
Or
redis-cli AUTH appuser strongapppassword
```

Step 3: Disable Dangerous Commands

Commands that can be weaponized if Redis is exposed:

```conf
In redis.conf. rename dangerous commands to empty string to disable
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command CONFIG ""
rename-command SHUTDOWN "SHUTDOWN_SAFE_KEY_9f3k2"
rename-command SLAVEOF ""
rename-command REPLICAOF ""
rename-command KEYS ""       # Use SCAN instead for production
```

If you need CONFIG in your application (some frameworks require it), rename it to a long random string instead of disabling it, and share only the renamed command with application code.

Step 4: Enable TLS (Redis 6+)

If Redis must communicate over a network (e.g., between app servers), use TLS:

```bash
Generate TLS certificates
mkdir -p /etc/redis/tls && cd /etc/redis/tls

CA
openssl req -new -x509 -days 3650 -nodes \
  -keyout ca.key -out ca.crt \
  -subj "/CN=Redis-CA"

Server cert
openssl req -new -nodes -keyout server.key -out server.csr \
  -subj "/CN=redis.internal"
openssl x509 -req -days 1825 \
  -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

chmod 600 /etc/redis/tls/*.key
chown redis:redis /etc/redis/tls/*
```

In redis.conf:
```conf
Disable plain port
port 0

Enable TLS
tls-port 6380
tls-cert-file /etc/redis/tls/server.crt
tls-key-file /etc/redis/tls/server.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients yes
tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
```

Connect with TLS:
```bash
redis-cli --tls \
  --cert /etc/redis/tls/client.crt \
  --key /etc/redis/tls/client.key \
  --cacert /etc/redis/tls/ca.crt \
  -p 6380 AUTH appuser strongapppassword
```

Step 5: Run as a Non-Root User

Redis should run as the `redis` system user, not root:

```bash
Verify service user
grep User /lib/systemd/system/redis-server.service
Should show: User=redis

If not, create override
sudo systemctl edit redis-server
Add:
[Service]
User=redis
Group=redis
```

Set file permissions:
```bash
sudo chown -R redis:redis /var/lib/redis /etc/redis
sudo chmod 750 /var/lib/redis
sudo chmod 640 /etc/redis/redis.conf /etc/redis/users.acl
```

Step 6: Memory and Resource Limits

Prevent Redis from consuming all available memory:

```conf
Maximum memory (adjust for your server)
maxmemory 512mb

Eviction policy when maxmemory is reached
allkeys-lru: evict least recently used keys (good for caches)
maxmemory-policy allkeys-lru

Prevent fork exhausting RAM during AOF rewrite
active-expire-enabled yes
lazyfree-lazy-eviction yes
```

Step 7: Persistence Security

Disable persistence if Redis is only used as a cache (reduces attack surface):

```conf
Disable RDB snapshots
save ""

Disable AOF (append-only file)
appendonly no
```

If you need persistence, set safe file paths:
```conf
dir /var/lib/redis
dbfilename dump.rdb
appendfilename "appendonly.aof"
```

Step 8: Network Firewall Rules

Even with `bind 127.0.0.1`, add UFW rules as defense-in-depth:

```bash
Block the default Redis port from all external sources
sudo ufw deny 6379/tcp
sudo ufw deny 6380/tcp

Only allow from specific app servers on private network
sudo ufw allow from 10.0.0.0/24 to any port 6380 proto tcp

sudo ufw reload
```

Verify Your Security

```bash
Test that unauthenticated access is blocked
redis-cli PING
Should return: NOAUTH Authentication required

Test that bound interfaces are correct
ss -tlnp | grep redis
Should show: 127.0.0.1:6379 only

Check ACL list
redis-cli AUTH admin adminpassword ACL LIST

Verify CONFIG is disabled (if renamed)
redis-cli AUTH appuser strongapppassword CONFIG GET maxmemory
Should return: ERR unknown command 'CONFIG'
```

Related Reading

- [Securing Docker Containers Best Practices](/securing-docker-containers-best-practices/)
- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)
- [Secure Elasticsearch Deployment Guide](/secure-elasticsearch-deployment-guide/)

- [Anonymous Domain Registration How To Buy Domain](/anonymous-domain-registration-how-to-buy-domain-without-expo/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [Best AI Tools for Container Security Scanning in Deployment](https://bestremotetools.com/best-ai-tools-for-container-security-scanning-in-deployment-/)
---

Related Articles

- [Securing Redis and MongoDB for Production](/securing-redis-mongodb-production-guide/)
- [How to Secure PostgreSQL for Production](/secure-postgresql-production-guide/)
- [GDPR Compliant User Authentication](/gdpr-compliant-user-authentication-design/)
- [Secure Shell Hardening Beyond SSH Config](/secure-shell-hardening-beyond-ssh-config/)
- [How to Secure NAS Storage for Home Use](/secure-nas-home-storage-guide/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
