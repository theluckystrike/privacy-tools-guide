---
layout: default
title: "How to Configure OpenPGP Key Server for Organization."
description: "A practical guide to setting up and configuring an internal OpenPGP key server for your organization. Covers server options, deployment, and best."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-configure-openpgp-key-server-for-organization-interna/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Managing PGP keys across an organization presents unique challenges. Public key servers like keys.openpgp.org serve the global community well, but many organizations require private key infrastructure for internal communications, code signing, and secure document exchange. This guide walks through configuring an internal OpenPGP key server tailored for organizational use.

## Why Run an Internal Key Server

Public key servers expose your key metadata to the internet. Your email address, key fingerprints, and signing history become searchable by anyone. Organizations in regulated industries or those handling sensitive data often cannot tolerate this exposure. An internal key server keeps your key infrastructure private while still providing the discovery and synchronization features that make PGP practical.

Internal key servers also solve the key freshness problem. When team members rotate keys or revoke compromised certificates, you need immediate propagation. Public servers may take hours or days to sync. An internal server provides instant key distribution within your network.

## Choosing Your Key Server Software

Three primary options exist for self-hosted OpenPGP key servers:

**SKS (Synchronizing Key Server)** is the traditional choice. Developed by the same team behind the GnuPG project, SKS pioneered the mesh synchronization model that public servers use. It requires a PostgreSQL database and handles key merging automatically.

**Hockeypuck** is a modern reimplementation written in Go. It offers a cleaner architecture, easier deployment, and compatibility with the SKS synchronization protocol. Hockeypuck works well for organizations wanting something simpler than SKS.

**Keys OpenPGP Key Server** is the server behind the popular keys.openpgp.org service. Written in Go, it supports modern features like keycloak authentication and WKD (Web Key Directory). However, it lacks the SKS synchronization protocol, making it less suitable if you plan to connect to the broader keyserver network.

For most organizations, Hockeypuck strikes the best balance between features and simplicity.

## Deploying Hockeypuck

Hockeypuck requires PostgreSQL and a Go runtime. Install dependencies on Ubuntu:

```bash
apt-get install postgresql golang-go
```

Create a PostgreSQL user and database:

```bash
su - postgres
psql -c "CREATE USER hockeypuck WITH PASSWORD 'your_secure_password';"
psql -c "CREATE DATABASE hockeypuck OWNER hockeypuck;"
```

Clone and build Hockeypuck:

```bash
git clone https://github.com/hockeypuck/hockeypuck.git
cd hockeypuck
make
```

Configure Hockeypuck by editing `hockeypuck.conf`. The essential settings include your database connection and the server's public-facing hostname:

```toml
[hockeypuck]
hostname = "keys.internal.yourcompany.com"
port = 11371

[database]
host = "localhost"
port = 5432
user = "hockeypuck"
password = "your_secure_password"
dbname = "hockeypuck"
```

Start the server:

```bash
./hockeypuck
```

The keyserver now runs on port 11371. Test it locally:

```bash
curl http://localhost:11371/pks/lookup?op=index&search=yourname@yourcompany.com
```

## Configuring Client Access

Your team needs to configure their GnuPG installations to query your internal server. Edit each user's `~/.gnupg/gpg.conf` or create a `dirmngr` configuration file:

```bash
mkdir -p ~/.gnupg
cat >> ~/.gnupg/dirmngr.conf <<EOF
keyserver hkps://keys.internal.yourcompany.com
keyserver-options timeout=10
EOF
```

Use `hkps` (HTTP over TLS) to encrypt traffic between clients and your server. You'll need to set up TLS certificates for your keyserver hostname. If you use self-signed certificates, distribute the CA certificate to client machines:

```bash
cat >> ~/.gnupg/dirmngr.conf <<EOF
tls-ca-file /path/to/your-ca-cert.pem
EOF
```

## Populating Your Key Server

Upload keys to your new server using GnuPG:

```bash
gpg --keyserver keys.internal.yourcompany.com --send-keys YOUR_KEY_ID
```

For bulk imports, Hockeypuck provides an import tool:

```bash
hockeypuck -import /path/to/keys.asc
```

You can also pull keys from the public network and mirror them internally. This is useful if your organization has keys on public servers that you want accessible without internet queries:

```bash
hockeypuck -sync yourname@yourcompany.com
```

## Setting Up Synchronization

If you run multiple key server nodes, configure SKS-style synchronization. Each node maintains a connection to peers and exchanges key updates. Add peers to your configuration:

```toml
[sks]
peers = [
    "keys2.internal.yourcompany.com",
    "keys.backup.yourcompany.com"
]
```

Synchronization uses port 11370 by default. Ensure firewall rules permit this traffic between your nodes but block external access.

## Automating Key Management

Organizations benefit from automated key expiration and rotation policies. Create a cron job that checks for expiring keys and sends notifications:

```bash
#!/bin/bash
# check-expiring-keys.sh
export GNUPGHOME=/path/to/admin/gnupg
gpg --keyserver keys.internal.yourcompany.com --list-keys | \
  gpg --keyserver keys.internal.yourcompany.com --check-sigs | \
  awk '/ expires/ { if ($8 <= 30) print $2 }' | \
  while read keyid; do
    echo "Key $keyid expires soon" | mail -s "Key Expiration Warning" admin@yourcompany.com
  done
```

Run this weekly to stay ahead of expiration issues.

## Security Considerations

Protect your key server like any critical infrastructure. Implement these measures:

**Network isolation**: Run your key server on an internal network segment. Only expose port 443 (if using TLS reverse proxy) to the VPN or intranet.

**Rate limiting**: Configure Hockeypuck to limit queries per IP address, preventing abuse:

```toml
[ratelimit]
requests_per_minute = 60
burst = 100
```

**Audit logging**: Enable detailed logging to track key lookups and uploads:

```toml
[logging]
level = "debug"
file = "/var/log/hockeypuck.log"
```

**Backup strategy**: Regularly export your PostgreSQL database. Test restoration procedures quarterly:

```bash
pg_dump -U hockeypuck hockeypuck > hockeypuck-backup-$(date +%Y%m%d).sql
```

## Troubleshooting Common Issues

**Keys not appearing in searches**: Verify your PostgreSQL database contains the keys. Check that your web server configuration serves the correct port. Review logs for indexing errors.

**Synchronization failures**: Confirm network connectivity between peers. Verify both servers use compatible versions. Check that firewall rules allow port 11370 traffic.

**Slow query performance**: Index the database properly. For large keyrings, consider adding a Redis cache layer for frequently queried keys.

## Conclusion

An internal OpenPGP key server provides your organization with private key infrastructure while maintaining the convenience of key discovery. Hockeypuck offers the easiest path to deployment, with PostgreSQL providing reliable storage. Configure TLS for client connections, automate key management tasks, and treat the server as critical infrastructure with proper backups and monitoring.

Once your key server runs, your team can securely exchange encrypted communications, sign code commits, and verify document signatures—all without exposing metadata to public servers.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
