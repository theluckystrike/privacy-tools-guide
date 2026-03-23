---
layout: default
title: "Secure API Key Rotation Automation Guide"
description: "Automate API key rotation for AWS, database, and third-party services using Vault, AWS Secrets Manager, and zero-downtime rotation scripts in Python and..."
date: 2026-03-22
author: theluckystrike
permalink: /secure-api-key-rotation-automation-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api, automation]
---

{% raw %}
# Secure API Key Rotation Automation Guide

Long-lived API keys are a silent security debt. A key that was legitimately issued three years ago may have been copied to ten different systems, a Slack message, and a committed `.env` file. Rotation doesn't just reduce the window after a compromise — it forces you to audit where each key is actually used. This guide automates rotation for AWS, database credentials, and third-party services.

## Why Manual Rotation Fails

Manual rotation has two failure modes:
1. It doesn't happen — keys are "rotated annually" but the policy slips
2. It causes downtime — the new key is deployed before old key is revoked, or vice versa

Automated rotation with zero-downtime overlap eliminates both.

---

## 1. AWS IAM Key Rotation

AWS IAM allows two access keys per user simultaneously. This enables overlap: create the new key, update all consumers, then delete the old key.

```python
#!/usr/bin/env python3
"""
rotate_iam_key.py — Zero-downtime IAM key rotation.
Usage: python3 rotate_iam_key.py <iam-username>
"""
import boto3, time, json, sys, subprocess
from datetime import datetime, timezone

USERNAME = sys.argv[1]
iam = boto3.client("iam")
ssm = boto3.client("ssm")  # or use Secrets Manager

def get_active_keys(username):
    resp = iam.list_access_keys(UserName=username)
    return [k for k in resp["AccessKeyMetadata"] if k["Status"] == "Active"]

def create_new_key(username):
    resp = iam.create_access_key(UserName=username)
    return resp["AccessKey"]

def update_ssm_parameter(key_id, key_secret):
    """Store new credentials in Systems Manager Parameter Store."""
    ssm.put_parameter(
        Name=f"/myapp/aws_access_key_id",
        Value=key_id,
        Type="SecureString",
        Overwrite=True,
    )
    ssm.put_parameter(
        Name=f"/myapp/aws_secret_access_key",
        Value=key_secret,
        Type="SecureString",
        Overwrite=True,
    )
    print(f"[+] Updated SSM parameters with new key {key_id[:8]}...")

def wait_for_propagation(seconds=15):
    """IAM key propagation can take up to 15 seconds globally."""
    print(f"[*] Waiting {seconds}s for IAM propagation...")
    time.sleep(seconds)

def verify_new_key(key_id, key_secret):
    """Verify the new key works before deleting the old one."""
    test_client = boto3.client(
        "sts",
        aws_access_key_id=key_id,
        aws_secret_access_key=key_secret,
        region_name="us-east-1",
    )
    identity = test_client.get_caller_identity()
    print(f"[+] New key verified: {identity['Arn']}")
    return True

def deactivate_key(username, key_id):
    iam.update_access_key(
        UserName=username,
        AccessKeyId=key_id,
        Status="Inactive",
    )
    print(f"[+] Deactivated old key {key_id[:8]}...")

def delete_key(username, key_id):
    iam.delete_access_key(UserName=username, AccessKeyId=key_id)
    print(f"[+] Deleted old key {key_id[:8]}...")

if __name__ == "__main__":
    existing_keys = get_active_keys(USERNAME)
    if len(existing_keys) >= 2:
        raise RuntimeError("User already has 2 active keys — clean up first")

    old_key_id = existing_keys[0]["AccessKeyId"] if existing_keys else None
    print(f"[*] Old key: {old_key_id[:8]}..." if old_key_id else "[*] No existing key")

    # Create new key
    new_key = create_new_key(USERNAME)
    print(f"[+] Created new key: {new_key['AccessKeyId'][:8]}...")

    # Store in Parameter Store
    update_ssm_parameter(new_key["AccessKeyId"], new_key["SecretAccessKey"])

    # Wait for propagation
    wait_for_propagation(15)

    # Verify
    if not verify_new_key(new_key["AccessKeyId"], new_key["SecretAccessKey"]):
        delete_key(USERNAME, new_key["AccessKeyId"])
        raise RuntimeError("New key verification failed — rolled back")

    # Deactivate old key (keep for 24h before deletion as safety net)
    if old_key_id:
        deactivate_key(USERNAME, old_key_id)
        print(f"[!] Old key {old_key_id[:8]}... deactivated but not yet deleted.")
        print(f"[!] Run with --delete-old flag after 24h to complete rotation.")
```

---

## 2. Database Password Rotation (PostgreSQL)

```bash
#!/bin/bash
# rotate_db_password.sh — rotate PostgreSQL app user password
# Runs as a privileged user (postgres or DBA account)

set -euo pipefail

APP_USER="myapp_user"
DB_HOST="127.0.0.1"
DB_PORT="5432"
DBA_USER="postgres"
VAULT_PATH="secret/myapp/db"

# Generate a strong random password
NEW_PASS=$(openssl rand -base64 48 | tr -d '/+=' | head -c 48)

echo "[*] Rotating password for ${APP_USER}"

# Update password in PostgreSQL
PGPASSWORD="$POSTGRES_MASTER_PASS" psql \
  -h "$DB_HOST" -p "$DB_PORT" -U "$DBA_USER" \
  -c "ALTER USER ${APP_USER} WITH PASSWORD '${NEW_PASS}';"

echo "[+] Password updated in PostgreSQL"

# Store new password in Vault
vault kv put "${VAULT_PATH}" password="${NEW_PASS}" rotated_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
echo "[+] Password stored in Vault at ${VAULT_PATH}"

# Verify new password works
PGPASSWORD="$NEW_PASS" psql \
  -h "$DB_HOST" -p "$DB_PORT" -U "$APP_USER" \
  -c "SELECT 1;" >/dev/null 2>&1 && echo "[+] New password verified" \
  || { echo "[!] New password verification FAILED"; exit 1; }

# Signal app to reload (depends on your deployment)
# For Kubernetes:
kubectl rollout restart deployment/myapp

echo "[+] Rotation complete"
```

---

## 3. Third-Party API Key Rotation (Stripe, SendGrid, etc.)

Most SaaS providers allow creating multiple API keys. Use this pattern:

```python
#!/usr/bin/env python3
"""
rotate_stripe_key.py — rotate Stripe restricted key
Requires Stripe API key with key management permissions.
"""
import stripe
import boto3
import json
from datetime import datetime

stripe.api_key = "sk_live_old_management_key"
secrets_client = boto3.client("secretsmanager", region_name="us-east-1")
SECRET_NAME = "prod/stripe/api_key"

def get_current_key_name() -> str:
    resp = secrets_client.get_secret_value(SecretId=SECRET_NAME)
    data = json.loads(resp["SecretString"])
    return data.get("key_name", "")

def create_new_stripe_key(old_key_name: str) -> dict:
    """Create a new restricted key with the same permissions as the old one."""
    new_name = f"app-key-{datetime.now().strftime('%Y%m')}"
    # Stripe API: create restricted key (via dashboard API or Stripe CLI)
    # This is a simplified example — actual implementation uses Stripe's key API
    return {
        "id": "rk_live_new_key_id",
        "secret": "rk_live_new_key_secret",
        "name": new_name
    }

def update_secret(new_key: dict):
    secrets_client.put_secret_value(
        SecretId=SECRET_NAME,
        SecretString=json.dumps({
            "key": new_key["secret"],
            "key_name": new_key["name"],
            "rotated_at": datetime.utcnow().isoformat(),
        })
    )
    print(f"[+] Updated Secrets Manager with key {new_key['name']}")

def verify_new_key(new_key_secret: str) -> bool:
    test_client = stripe.StripeClient(new_key_secret)
    try:
        test_client.balance.retrieve()
        return True
    except stripe.AuthenticationError:
        return False

def delete_old_key(key_id: str):
    # Stripe: delete restricted key via API
    print(f"[+] Deleted old Stripe key {key_id}")

old_key_name = get_current_key_name()
new_key = create_new_stripe_key(old_key_name)
update_secret(new_key)

if not verify_new_key(new_key["secret"]):
    raise RuntimeError("New Stripe key verification failed")

print("[+] Verified new key works")
# Allow 5 minutes for all services to pick up new key
# Then delete old key
```

---

## 4. Automated Rotation with HashiCorp Vault

Vault's dynamic secrets engine generates credentials on demand and revokes them automatically:

```bash
# Configure Vault to rotate a static secret on a schedule
vault write secret/myapp/api_key \
  key="current_api_key" \
  rotation_period="168h"  # rotate weekly

# Use Vault Agent to write credentials to a file automatically renewed
cat > /etc/vault-agent.hcl <<'EOF'
vault { address = "https://vault.internal:8200" }

auto_auth {
  method "aws" {
    config = { role = "myapp-rotation" }
  }
}

template {
  contents = <<TMPL
{{ with secret "secret/myapp/api_key" }}{{ .Data.data.key }}{{ end }}
TMPL
  destination = "/run/secrets/api_key"
  perms       = 0600
  command     = "systemctl reload myapp"
}
EOF

vault agent -config=/etc/vault-agent.hcl -daemon
```

---

## 5. Rotation Schedule and Monitoring

```bash
# /etc/cron.d/key-rotation
# Rotate IAM keys weekly, database passwords monthly
0 2 * * 0 ubuntu /opt/scripts/rotate_iam_key.py myapp-user >> /var/log/rotation.log 2>&1
0 3 1 * * ubuntu /opt/scripts/rotate_db_password.sh >> /var/log/rotation.log 2>&1

# Alert on rotation failures
# Send to Slack or PagerDuty if any rotation script exits non-zero
```

```python
# Add to each rotation script — notify on failure
import requests, os

def alert_on_failure(service: str, error: str):
    webhook = os.environ.get("SLACK_WEBHOOK_URL")
    if webhook:
        requests.post(webhook, json={
            "text": f":rotating_light: *Key rotation FAILED* for `{service}`\n```{error}```"
        })
```

---

## Key Rotation Audit Log

```bash
# Query CloudTrail for IAM key usage after deactivation
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time "2026-03-15" \
  --query 'Events[].{Time:EventTime,Name:EventName,Source:EventSource}' \
  --output table

# If old key is used after deactivation — something didn't get updated
```

---

## Rotation Checklist

- [ ] Keys have a documented rotation schedule (IAM: 90d max, DB: 30d, third-party: 180d)
- [ ] Vault or Secrets Manager used — no plaintext keys in files or env
- [ ] Rotation creates new key before revoking old (zero-downtime overlap)
- [ ] Rotation scripts verify new key before deleting old key
- [ ] Failed rotation triggers an alert (Slack, PagerDuty)
- [ ] All key usages inventoried — rotation fails if a consumer isn't updated
- [ ] CloudTrail / audit logs reviewed after rotation for any missed consumers

---

## Related Reading

- [Secure Environment Variable Management](/secure-environment-variable-management-guide/)
- [Secure API Gateway Setup with Kong](/kong-api-gateway-secure-setup-guide/)
- [Secure JWT Implementation Best Practices](/secure-jwt-implementation-best-practices/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
