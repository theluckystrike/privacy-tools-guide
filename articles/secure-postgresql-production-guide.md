---
layout: default
title: "How to Secure PostgreSQL for Production"
description: "Harden a PostgreSQL server with network restrictions, role-based access, SSL enforcement, audit logging, and encrypted backups for production deployments"
date: 2026-03-22
author: theluckystrike
permalink: /secure-postgresql-production-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Secure PostgreSQL for Production

A default PostgreSQL installation is not production-ready from a security perspective. The superuser is trusted locally without a password, network access may be open, and logging is minimal. This guide takes a fresh PostgreSQL 16 install and makes it fit for production.

## Key Takeaways

- **Topics covered**: step 1: lock down network access, step 2: change the default superuser password, step 3: enforce ssl/tls
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## Step 1: Lock Down Network Access

```bash
# postgresql.conf — restrict which interfaces PostgreSQL listens on
# Default is localhost only, but verify:
grep listen_addresses /etc/postgresql/16/main/postgresql.conf
# listen_addresses = 'localhost'   (good — only listen locally)

# If your app is on the same host, localhost is correct.
# If you need remote connections, list specific IPs:
# listen_addresses = '192.168.1.50, 10.0.0.50'
# Never: listen_addresses = '*'  (exposes to all interfaces)
```

```bash
# pg_hba.conf — control who can connect
# File location:
sudo -u postgres psql -c "SHOW hba_file;"

# Secure pg_hba.conf (example)
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

```
# pg_hba.conf — secure configuration
# TYPE  DATABASE        USER            ADDRESS         METHOD

# Superuser: local socket only, not from network
local   all             postgres                        peer

# Application user: only from specific IP, require password, require SSL
hostssl appdb           appuser         10.0.0.0/24     scram-sha-256

# Deny all other connections
host    all             all             0.0.0.0/0       reject
host    all             all             ::/0            reject
```

```bash
# Reload after editing
sudo systemctl reload postgresql
```

## Step 2: Change the Default Superuser Password

```bash
sudo -u postgres psql

-- Set a strong password for the postgres superuser
\password postgres

-- Or via psql one-liner:
sudo -u postgres psql -c "\password postgres"
```

Better: disable remote login for the postgres user entirely (the `peer` auth in pg_hba.conf above already does this for local socket connections from any user other than the `postgres` OS user).

## Step 3: Enforce SSL/TLS

```bash
# Enable SSL in postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf
```

```
ssl = on
ssl_cert_file = '/etc/ssl/certs/postgresql.crt'
ssl_key_file = '/etc/ssl/private/postgresql.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'      # for client cert verification
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'HIGH:!aNULL:!MD5'
```

```bash
# Generate a self-signed certificate (or use your CA)
openssl req -new -x509 -days 3650 -nodes \
  -out /etc/ssl/certs/postgresql.crt \
  -keyout /etc/ssl/private/postgresql.key \
  -subj "/CN=postgres.internal"

sudo chown postgres:postgres /etc/ssl/private/postgresql.key
sudo chmod 600 /etc/ssl/private/postgresql.key

# Force SSL for the application connection
# In pg_hba.conf: use "hostssl" instead of "host"
# This rejects plaintext connections from the application

sudo systemctl restart postgresql
```

```bash
# Verify SSL is working
psql "host=localhost dbname=appdb user=appuser sslmode=require" -c "\conninfo"
# SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384)
```

## Step 4: Create Least-Privilege Roles

```sql
-- Connect as superuser
sudo -u postgres psql

-- Create a dedicated application role with no superuser privileges
CREATE ROLE appuser WITH LOGIN PASSWORD 'strong-random-password-here';

-- Create the database owned by a separate owner role
CREATE ROLE appowner NOLOGIN;
CREATE DATABASE appdb OWNER appowner;

-- Connect to the application database
\c appdb

-- Grant only what the application needs
-- Typical CRUD application:
GRANT CONNECT ON DATABASE appdb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;

-- Future tables and sequences inherit permissions
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO appuser;

-- Read-only replica user
CREATE ROLE readuser WITH LOGIN PASSWORD 'another-strong-password';
GRANT CONNECT ON DATABASE appdb TO readuser;
GRANT USAGE ON SCHEMA public TO readuser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO readuser;
```

```sql
-- Revoke public schema creation from PUBLIC (default in Postgres 14 and older)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Verify privileges
\dp   -- show table privileges
\du   -- show users and roles
```

## Step 5: Enable Audit Logging

```bash
# postgresql.conf — logging configuration
log_destination = 'syslog'
syslog_facility = 'LOCAL0'
syslog_ident = 'postgres'

# What to log
log_connections = on
log_disconnections = on
log_failed_attempts = on
log_hostname = off     # avoid DNS lookups in logs

# Log slow queries (useful for both performance and detecting attacks)
log_min_duration_statement = 1000   # log queries slower than 1 second

# Log all DDL (CREATE, DROP, ALTER)
log_statement = 'ddl'

# Log all connections from IP ranges you do not expect
# (use pg_hba.conf to deny them, but log helps detect attempts)
```

For more detailed auditing, install `pgaudit`:

```bash
# Install pgaudit
sudo apt install postgresql-16-pgaudit

# postgresql.conf
shared_preload_libraries = 'pgaudit'

# pgaudit settings
pgaudit.log = 'ddl, write, role'
pgaudit.log_catalog = off
pgaudit.log_parameter = on
pgaudit.log_statement_once = off
```

```sql
-- Enable pgaudit per role (audit all writes by the app user)
ALTER ROLE appuser SET pgaudit.log = 'write';
```

## Step 6: Restrict Row-Level Access

Row-Level Security (RLS) ensures users can only see their own data even if they access the table:

```sql
-- Enable RLS on a multi-tenant table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create a policy: each user sees only their own orders
CREATE POLICY user_orders ON orders
  USING (user_id = current_setting('app.current_user_id')::int);

-- Application sets this session variable per request
-- In your app code:
-- SET LOCAL app.current_user_id = '42';
-- SELECT * FROM orders;  -- returns only orders for user 42
```

## Step 7: Encrypted Backups

```bash
#!/bin/bash
# pg-backup-encrypted.sh

DB_NAME="appdb"
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y-%m-%d-%H%M)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}-${DATE}.sql.gz.age"
RECIPIENTS_FILE="/root/.age-recipients"   # age public keys

mkdir -p "$BACKUP_DIR"

# Dump and encrypt in a pipeline — plaintext never touches disk
pg_dump -U postgres "$DB_NAME" \
  | gzip \
  | age -R "$RECIPIENTS_FILE" \
  > "$BACKUP_FILE"

echo "Backup written: $BACKUP_FILE ($(du -sh "$BACKUP_FILE" | cut -f1))"

# Keep 30 days of backups
find "$BACKUP_DIR" -name "*.age" -mtime +30 -delete
```

```bash
# Restore
age -d -i ~/.age/identity.txt backup.sql.gz.age | gunzip | psql -U postgres appdb
```

## Step 8: Rotate Credentials

```bash
# Script to rotate the application user password and update the secret manager
NEW_PASSWORD=$(openssl rand -base64 32)

sudo -u postgres psql -c "ALTER ROLE appuser PASSWORD '${NEW_PASSWORD}';"

# Update in your secret manager (example: AWS)
aws secretsmanager put-secret-value \
  --secret-id "prod/appdb/appuser" \
  --secret-string "{\"password\":\"${NEW_PASSWORD}\"}"

echo "Password rotated. Restart application to pick up new credentials."
```

## Related Reading

- [How to Set Up Vault for Secrets Management](/privacy-tools-guide/hashicorp-vault-secrets-management-setup/)
- [Secure API Key Management for Developers](/privacy-tools-guide/secure-api-key-management-developers/)
- [How to Secure Your GitHub Account](/privacy-tools-guide/secure-github-account-hardening-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
