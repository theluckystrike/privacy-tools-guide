---






















































































































































































































































































































































































layout: default
title: "Secure Database Connection Pooling Guide"
<<<<<<< HEAD
description: "Configure PgBouncer and HikariCP with TLS, least-privilege credentials, connection limits, and auth passthrough to protect PostgreSQL from credential."
=======
description: "Configure PgBouncer and HikariCP with TLS, least-privilege credentials, connection limits, and auth passthrough to protect PostgreSQL from credential
>>>>>>> 900910b9245b9b701f9371a2b27423efb875d01e
date: 2026-03-22
author: "Privacy Tools Guide"
permalink: /secure-database-connection-pooling-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---























































































































































































































































































































































































{% raw %}
# Secure Database Connection Pooling Guide

Database connection poolers sit between your application and PostgreSQL (or MySQL), multiplexing thousands of app connections onto a smaller pool of actual database connections. Without secure configuration, a pooler becomes a chokepoint for credential theft and privilege escalation. This guide covers PgBouncer (the standard PostgreSQL pooler) and HikariCP (Java/JVM) with full TLS and auth hardening.

## Why Pooler Security Matters

- Poolers often run with static credentials — if compromised, every tenant's connection is exposed
- Unencrypted pooler-to-database links can be sniffed on shared infrastructure
- Over-privileged pooler credentials allow attackers to `DROP TABLE` across schemas
- Connection limits protect against connection-exhaustion DoS attacks

---

## Part 1: PgBouncer Setup

### Install PgBouncer

```bash
sudo apt install -y pgbouncer

# Verify
pgbouncer --version
```

---

### 1. Generate TLS Certificates for Pooler-to-Database Link

```bash
# On the PostgreSQL server — create client cert for PgBouncer
openssl genrsa -out pgbouncer-client-key.pem 2048
openssl req -new -key pgbouncer-client-key.pem \
  -subj "/CN=pgbouncer-client" \
  -out pgbouncer-client.csr

# Sign with your CA
openssl x509 -req -in pgbouncer-client.csr \
  -CA /etc/ssl/pgca/ca.crt \
  -CAkey /etc/ssl/pgca/ca.key \
  -CAcreateserial \
  -days 365 -out pgbouncer-client-cert.pem

# TLS for app-to-PgBouncer
openssl genrsa -out pgbouncer-server-key.pem 2048
openssl req -new -key pgbouncer-server-key.pem \
  -subj "/CN=pgbouncer.internal" \
  -out pgbouncer-server.csr
# ... sign same way
```

---

### 2. Configure PgBouncer

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# Map virtual database names to real databases
myapp_db = host=127.0.0.1 port=5432 dbname=myapp \
           user=myapp_pool_user \
           client_encoding=UTF8

# Read-only replica pool
myapp_ro = host=replica.internal port=5432 dbname=myapp \
           user=myapp_ro_user

[pgbouncer]
# Network
listen_addr = 127.0.0.1   # never 0.0.0.0 unless behind firewall
listen_port = 5432

# TLS — app-to-pgbouncer
client_tls_sslmode = require
client_tls_key_file = /etc/pgbouncer/pgbouncer-server-key.pem
client_tls_cert_file = /etc/pgbouncer/pgbouncer-server-cert.pem
client_tls_ca_file = /etc/ssl/pgca/ca.crt

# TLS — pgbouncer-to-postgres
server_tls_sslmode = verify-full
server_tls_key_file = /etc/pgbouncer/pgbouncer-client-key.pem
server_tls_cert_file = /etc/pgbouncer/pgbouncer-client-cert.pem
server_tls_ca_file = /etc/ssl/pgca/ca.crt

# Pooling mode
pool_mode = transaction   # transaction-level pooling (best throughput)
# session pooling is required if using SET LOCAL, prepared statements, advisory locks

# Pool sizes — tune based on DB server capacity
default_pool_size = 20   # max server connections per database
max_client_conn = 500    # max app connections to PgBouncer
reserve_pool_size = 5    # emergency reserve

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Admin
admin_users = pgbouncer_admin
stats_users = monitoring

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 30
query_wait_timeout = 120
```

---

### 3. Create the User List (Hashed Passwords)

```bash
# Generate SCRAM-SHA-256 hash for a user
# In psql on PostgreSQL server:
psql -c "SELECT concat('\"', usename, '\" \"', passwd, '\"') FROM pg_shadow WHERE usename='myapp_pool_user';"

# Paste the output into userlist.txt
sudo tee /etc/pgbouncer/userlist.txt > /dev/null <<'EOF'
"myapp_pool_user" "SCRAM-SHA-256$4096:salt_base64_here$stored_key_here:server_key_here"
"myapp_ro_user"   "SCRAM-SHA-256$..."
"pgbouncer_admin" "SCRAM-SHA-256$..."
EOF

sudo chmod 600 /etc/pgbouncer/userlist.txt
sudo chown pgbouncer: /etc/pgbouncer/userlist.txt
```

---

### 4. Least-Privilege Database Users

```sql
-- On PostgreSQL: create a pool user with minimal privileges
CREATE USER myapp_pool_user WITH PASSWORD 'strong_password_here';
GRANT CONNECT ON DATABASE myapp TO myapp_pool_user;
GRANT USAGE ON SCHEMA public TO myapp_pool_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp_pool_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO myapp_pool_user;

-- Explicitly revoke dangerous operations
REVOKE CREATE ON SCHEMA public FROM myapp_pool_user;
REVOKE DROP, TRUNCATE ON ALL TABLES IN SCHEMA public FROM myapp_pool_user;

-- Read-only user for analytics queries
CREATE USER myapp_ro_user WITH PASSWORD 'another_strong_password';
GRANT CONNECT ON DATABASE myapp TO myapp_ro_user;
GRANT USAGE ON SCHEMA public TO myapp_ro_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO myapp_ro_user;
```

---

### 5. Start and Monitor PgBouncer

```bash
sudo systemctl enable --now pgbouncer
sudo systemctl status pgbouncer

# Connect to PgBouncer admin console
psql -h 127.0.0.1 -p 5432 -U pgbouncer_admin pgbouncer

# Admin console commands:
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW STATS;
SHOW CONFIG;

# Check current connection counts
SHOW POOLS;
# database | user | cl_active | cl_waiting | sv_active | sv_idle | maxwait
```

---

## Part 2: HikariCP (Java/Spring Boot)

HikariCP is the default connection pool for Spring Boot since 2.0.

### Secure Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://pgbouncer.internal:5432/myapp_db?sslmode=verify-full\
         &sslrootcert=/app/certs/ca.crt\
         &sslcert=/app/certs/client-cert.pem\
         &sslkey=/app/certs/client-key.pem
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      pool-name: myapp-pool
      # Sizing
      maximum-pool-size: 20
      minimum-idle: 5
      # Timeouts
      connection-timeout: 30000      # 30s to acquire connection
      idle-timeout: 600000           # 10m idle before removal
      max-lifetime: 1800000          # 30m max lifetime (rotate before Vault TTL)
      keepalive-time: 300000         # 5m heartbeat query
      # Validation
      connection-test-query: SELECT 1
      validation-timeout: 5000
      # Leak detection
      leak-detection-threshold: 60000  # warn if connection held >60s
```

### Read Credentials from Vault at Startup

```java
@Configuration
public class DataSourceConfig {

    @Autowired
    private VaultOperations vault;

    @Bean
    @Primary
    public DataSource dataSource() {
        // Fetch dynamic credentials from Vault
        VaultResponseSupport<Map<String, Object>> response =
            vault.read("database/creds/myapp");

        String username = (String) response.getData().get("username");
        String password = (String) response.getData().get("password");

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://pgbouncer.internal:5432/myapp_db"
            + "?sslmode=verify-full&sslrootcert=/app/certs/ca.crt");
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setMaxLifetime(3600000);  // 1h — must be less than Vault TTL

        return new HikariDataSource(config);
    }
}
```

---

## Part 3: Connection Limit Enforcement at PostgreSQL

Even with a pooler, set limits at the database layer:

```sql
-- Limit connections per user
ALTER USER myapp_pool_user CONNECTION LIMIT 25;
ALTER USER myapp_ro_user   CONNECTION LIMIT 10;

-- Per-database limit
ALTER DATABASE myapp CONNECTION LIMIT 100;

-- View current limits
SELECT datname, datconnlimit FROM pg_database;
SELECT usename, connlimit FROM pg_user;

-- Monitor active connections
SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename;
```

---

## Security Checklist

- [ ] PgBouncer `listen_addr` is `127.0.0.1` or private network only
- [ ] TLS enforced on both app-to-pooler and pooler-to-database links
- [ ] `auth_type = scram-sha-256` (not `trust` or `md5`)
- [ ] `userlist.txt` owned by pgbouncer, mode 600
- [ ] Pool users have no CREATE/DROP/TRUNCATE privileges
- [ ] `max_client_conn` prevents connection exhaustion from rogue clients
- [ ] HikariCP `max-lifetime` less than Vault dynamic secret TTL
- [ ] `leak-detection-threshold` set to catch connection leaks early
- [ ] Database-level `CONNECTION LIMIT` set per user and database

---

## Related Reading

- [Secure Environment Variable Management](/privacy-tools-guide/secure-environment-variable-management-guide/)
- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [How to Set Up Wazuh SIEM for Small Teams](/privacy-tools-guide/wazuh-siem-small-teams-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
