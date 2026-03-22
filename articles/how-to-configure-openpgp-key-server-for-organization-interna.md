---
layout: default
title: "How To Configure Openpgp Key Server For Organization"
description: "A practical guide to setting up and configuring an internal OpenPGP key server for your organization. Covers server options, deployment, and best"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-configure-openpgp-key-server-for-organization-interna/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Managing PGP keys across an organization presents unique challenges. Public key servers like keys.openpgp.org serve the global community well, but many organizations require private key infrastructure for internal communications, code signing, and secure document exchange. This guide walks through configuring an internal OpenPGP key server tailored for organizational use.

## Key Takeaways

- **Upload to $INTERNAL_KEYSERVER**: gpg --keyserver $INTERNAL_KEYSERVER --send-keys [NEW_KEY_ID]
3.
- **Add peers to your configuration**: ```toml
[sks]
peers = [
    "keys2.internal.yourcompany.com",
    "keys.backup.yourcompany.com"
]
```

Synchronization uses port 11370 by default.
- **Verify both servers use**: compatible versions.
- **Update team wiki with**: new key ID Timeline: Complete rotation at least 7 days before expiration.
- **This guide walks through**: configuring an internal OpenPGP key server tailored for organizational use.
- **Developed by the same**: team behind the GnuPG project, SKS pioneered the mesh synchronization model that public servers use.

## Table of Contents

- [Why Run an Internal Key Server](#why-run-an-internal-key-server)
- [Prerequisites](#prerequisites)
- [Security Considerations](#security-considerations)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Advanced Key Management Workflows](#advanced-key-management-workflows)
- [Integration with Code Signing Requirements](#integration-with-code-signing-requirements)
- [Monitoring and Compliance Auditing](#monitoring-and-compliance-auditing)

## Why Run an Internal Key Server

Public key servers expose your key metadata to the internet. Your email address, key fingerprints, and signing history become searchable by anyone. Organizations in regulated industries or those handling sensitive data often cannot tolerate this exposure. An internal key server keeps your key infrastructure private while still providing the discovery and synchronization features that make PGP practical.

Internal key servers also solve the key freshness problem. When team members rotate keys or revoke compromised certificates, you need immediate propagation. Public servers may take hours or days to sync. An internal server provides instant key distribution within your network.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Choose Your Key Server Software

Three primary options exist for self-hosted OpenPGP key servers:

**SKS (Synchronizing Key Server)** is the traditional choice. Developed by the same team behind the GnuPG project, SKS pioneered the mesh synchronization model that public servers use. It requires a PostgreSQL database and handles key merging automatically.

**Hockeypuck** is a modern reimplementation written in Go. It offers a cleaner architecture, easier deployment, and compatibility with the SKS synchronization protocol. Hockeypuck works well for organizations wanting something simpler than SKS.

**Keys OpenPGP Key Server** is the server behind the popular keys.openpgp.org service. Written in Go, it supports modern features like keycloak authentication and WKD (Web Key Directory). However, it lacks the SKS synchronization protocol, making it less suitable if you plan to connect to the broader keyserver network.

For most organizations, Hockeypuck strikes the best balance between features and simplicity.

### Step 2: Deploy Hockeypuck

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

### Step 3: Configure Client Access

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

### Step 4: Populating Your Key Server

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

### Step 5: Set Up Synchronization

If you run multiple key server nodes, configure SKS-style synchronization. Each node maintains a connection to peers and exchanges key updates. Add peers to your configuration:

```toml
[sks]
peers = [
    "keys2.internal.yourcompany.com",
    "keys.backup.yourcompany.com"
]
```

Synchronization uses port 11370 by default. Ensure firewall rules permit this traffic between your nodes but block external access.

### Step 6: Automate Key Management

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

## Advanced Key Management Workflows

Beyond basic key distribution, implement organizational policies:

**Automated Key Rotation**: Enforce regular key rotation to limit damage from compromised keys:

```bash
#!/bin/bash
# rotate_organizational_keys.sh
# Run monthly to identify expiring keys and prepare rotations

EXPIRATION_DAYS=90
INTERNAL_KEYSERVER="keys.internal.yourcompany.com"

# Export all keys with expiration info
gpg --keyserver $INTERNAL_KEYSERVER --list-keys --with-colons | \
  awk -F: '$1=="pub" && $7=="0" {print $5}' | \
  while read keyid; do
    # Get key owner email
    owner=$(gpg --keyid-format LONG -k $keyid | grep uid | head -1 | awk -F'<' '{print $2}' | tr -d '>')

    # Calculate days until expiration
    exp_date=$(gpg --with-colons --list-keys $keyid | grep "^pub:" | cut -d: -f7)
    exp_unix=$exp_date
    current_unix=$(date +%s)
    days_until_exp=$(( ($exp_unix - $current_unix) / 86400 ))

    if [ $days_until_exp -lt $EXPIRATION_DAYS ] && [ $days_until_exp -gt 0 ]; then
      echo "KEY ROTATION REQUIRED: $owner ($keyid) expires in $days_until_exp days"

      # Send reminder to employee
      mail -s "Action Required: Rotate Your PGP Key" "$owner" <<EOF
Your PGP key ($keyid) will expire in $days_until_exp days.

To rotate:
1. Create new key: gpg --gen-key
2. Upload to $INTERNAL_KEYSERVER: gpg --keyserver $INTERNAL_KEYSERVER --send-keys [NEW_KEY_ID]
3. Sign your new key with old key to establish continuity
4. Update team wiki with new key ID

Timeline: Complete rotation at least 7 days before expiration.
EOF
    fi
done
```

Schedule this script monthly via cron. Proactive rotation prevents operational incidents when a key expires and developers lose the ability to sign commits.

**Key Signing Ceremony**: For high-security environments, implement key signing ceremonies where team members sign each other's keys in person:

```bash
#!/bin/bash
# key_signing_ceremony.sh
# Formal ceremony for establishing web of trust

echo "=== PGP Key Signing Ceremony ==="
echo "Purpose: Establish organizational web of trust"
echo "Participants must have valid ID and fingerprint verification"
echo ""

# Step 1: Import keys
echo "Step 1: Import all participating keys"
for participant in alice@company.com bob@company.com charlie@company.com; do
  gpg --keyserver keys.internal.yourcompany.com --recv-keys $participant
done

# Step 2: Verify fingerprints in person
echo "Step 2: Verify fingerprints (each person reads out loud)"
for participant in alice@company.com bob@company.com charlie@company.com; do
  gpg --fingerprint $participant
done

# Step 3: Sign each key
echo "Step 3: Sign each key if verification successful"
read -p "Verify all fingerprints? (y/n) " verified

if [ "$verified" = "y" ]; then
  for keyid in $(gpg --list-keys --with-colons | grep "^pub:" | cut -d: -f5); do
    gpg --sign-key $keyid
  done

  # Step 4: Upload signed keys back to server
  echo "Step 4: Upload signed keys"
  gpg --keyserver keys.internal.yourcompany.com --send-keys $keyid
fi

echo "Key signing ceremony complete."
```

This creates a web of trust where employees have personally verified each other's keys. It's slower to scale but creates extremely strong trust relationships.

## Integration with Code Signing Requirements

Organizations with compliance requirements often mandate code signing. Configure your key server for this:

**Git Configuration for Signed Commits**:

```bash
# Set user signing key globally
git config --global user.signingkey [YOUR_KEY_ID]

# Enable signing by default for commits
git config --global commit.gpgsign true

# Enable signing by default for tags
git config --global tag.gpgsign true

# Configure GPG to use your internal keyserver
cat >> ~/.gnupg/gpg.conf <<EOF
keyserver hkps://keys.internal.yourcompany.com
keyserver-options timeout=10
EOF
```

**Verification on CI/CD Pipeline**:

```yaml
# .github/workflows/verify-commits.yml
name: Verify Commit Signatures

on: [push, pull_request]

jobs:
  verify-signatures:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Need full history to check all commits

      - name: Configure internal keyserver
        run: |
          mkdir -p ~/.gnupg
          echo "keyserver hkps://keys.internal.yourcompany.com" >> ~/.gnupg/gpg.conf
          echo "keyserver-options timeout=10" >> ~/.gnupg/gpg.conf

      - name: Verify commit signatures
        run: |
          for commit in $(git log --pretty=%H origin/main..HEAD); do
            if git verify-commit $commit 2>/dev/null; then
              echo "✓ $commit is signed correctly"
            else
              echo "✗ $commit is NOT signed"
              exit 1
            fi
          done
```

This ensures every commit in production branches is cryptographically signed by an authorized developer. It prevents unauthorized code from being merged.

### Step 7: Disaster Recovery and Key Escrow

Organizations must plan for scenarios where key owners are unavailable:

**Key Escrow Procedure**: Securely backup keys in case of emergencies:

```bash
#!/bin/bash
# escrow_key.sh - Escrow procedure for organizational keys

# Step 1: Export private key (HIGHLY SENSITIVE)
gpg --armor --export-secret-key [KEY_ID] > /tmp/key_escrow.asc

# Step 2: Encrypt with 3-of-5 Shamir Secret Sharing
# Requires: ssss (Secret Sharing Scheme)
# brew install ssss (macOS) or apt-get install ssss (Ubuntu)

ssss-split -t 3 -n 5 -w escrow < /tmp/key_escrow.asc

# This produces 5 shares; any 3 can reconstruct the key
# Distribute shares to:
# 1. HR department
# 2. Finance department
# 3. Legal department
# 4. CTO
# 5. Operations director

# Store each share in a separate secure location
echo "Escrow shares generated. Distribute accordingly."

# Cleanup
shred -vfz -n 3 /tmp/key_escrow.asc
```

Using Shamir's Secret Sharing ensures no single person can recover a key unilaterally. You need 3 of the 5 shares, preventing any individual from accessing escrowed keys without accountability.

**Periodic Testing**: Quarterly, practice key recovery to ensure procedures work:

```bash
# Quarterly drill: Recover test key from escrow
# Only proceed if authorized by management

recovered_key=$(ssss-combine -w escrow < combined_shares.txt)
gpg --import $recovered_key

# Verify key functions
gpg --list-keys [RECOVERED_KEY_ID]
gpg --sign --trust-model always -u [RECOVERED_KEY_ID] /tmp/test_file.txt

# Verify signature works
gpg --verify /tmp/test_file.txt.gpg

# Clean up test materials
shred -vfz -n 3 /tmp/test_file.txt.gpg
```

Document that recovery works. This prevents discovering during an actual emergency that your escrow procedure is broken.

## Monitoring and Compliance Auditing

Implement continuous monitoring of your key server:

```python
import sqlite3
from datetime import datetime, timedelta
import logging

class KeyServerAudit:
    def __init__(self, db_path, log_path):
        self.db = sqlite3.connect(db_path)
        self.logger = logging.getLogger('keyserver')
        self.logger.addHandler(logging.FileHandler(log_path))

    def audit_key_distribution(self):
        """Ensure all employees have keys"""
        cursor = self.db.cursor()

        # Get all active employees
        cursor.execute("SELECT email FROM employees WHERE status='active'")
        employees = set(row[0] for row in cursor.fetchall())

        # Get all keys on server
        cursor.execute("SELECT uid FROM keys")
        keys = set(row[0] for row in cursor.fetchall())

        missing_keys = employees - keys
        if missing_keys:
            self.logger.warning(f"Missing keys for: {', '.join(missing_keys)}")

        return missing_keys

    def audit_key_expiration(self):
        """Check for expiring keys"""
        cursor = self.db.cursor()
        now = datetime.utcnow()
        thirty_days = now + timedelta(days=30)

        cursor.execute(
            "SELECT uid, expires FROM keys WHERE expires < ?",
            (thirty_days.timestamp(),)
        )

        expiring = cursor.fetchall()
        for uid, exp_time in expiring:
            days = (exp_time - now.timestamp()) / 86400
            self.logger.warning(f"Key {uid} expires in {days:.0f} days")

        return expiring

    def audit_access_logs(self):
        """Review who accessed which keys"""
        cursor = self.db.cursor()
        yesterday = datetime.utcnow() - timedelta(days=1)

        cursor.execute(
            "SELECT user, key_accessed, timestamp FROM access_log WHERE timestamp > ?",
            (yesterday.timestamp(),)
        )

        accesses = cursor.fetchall()
        for user, key, ts in accesses:
            self.logger.info(f"{user} accessed {key} at {datetime.fromtimestamp(ts)}")

        return accesses

# Usage
audit = KeyServerAudit('/var/lib/hockeypuck/hockeypuck.db', '/var/log/keyserver-audit.log')
audit.audit_key_distribution()
audit.audit_key_expiration()
audit.audit_access_logs()
```

Run this daily. Set up alerts for missing keys or expiring keys. Review access logs weekly to detect unusual patterns.

## Frequently Asked Questions

**How long does it take to configure openpgp key server for organization?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Self-Hosted Private Git Server with Gitea](/privacy-tools-guide/private-git-server-gitea-setup-guide/)
- [How to Check If Your Email Server Has Been Blacklisted](/privacy-tools-guide/how-to-check-if-your-email-server-has-been-blacklisted/)
- [VPN Provider Server Infrastructure How To Evaluate](/privacy-tools-guide/vpn-provider-server-infrastructure-how-to-evaluate-trustworthiness/)
- [Set Up Mail In A Box Private Email Server Complete 2026](/privacy-tools-guide/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [SSH Server Hardening Guide](/privacy-tools-guide/ssh-server-hardening-guide/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
```
{% endraw %}
