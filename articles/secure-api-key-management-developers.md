---
layout: default
title: "Secure API Key Management for Developers"
description: "Manage API keys safely with environment injection, secret managers, rotation automation, and detection of accidentally committed credentials"
date: 2026-03-22
author: theluckystrike
permalink: /secure-api-key-management-developers/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, api]
---

{% raw %}

Secure API Key Management for Developers

Leaked API keys cause real breaches. GitHub's secret scanning detects millions of exposed credentials per year. Most of them are committed accidentally in `.env` files, hardcoded in source, or left in shell history. This guide covers how to handle keys properly from development through production.

The Core Rule - Never Store Keys in Code

API keys do not belong in:
- Source code files
- `.env` files committed to git
- Shell history
- CI/CD log output
- Docker images

They belong in:
- Environment variables (injected at runtime)
- Secret managers (Vault, AWS Secrets Manager, 1Password Secrets Automation)
- `.env` files that are `.gitignore`d

Setting Up `.gitignore` Correctly

```bash
.gitignore. add these before any keys ever get committed
.env
.env.*
!.env.example
*.pem
*.key
*_rsa
*_ed25519
credentials.json
service-account.json
*secret*
*credentials*
```

Create a `.env.example` with placeholder values that goes into the repo:

```bash
.env.example. commit this
DATABASE_URL=postgres://user:password@localhost:5432/mydb
STRIPE_SECRET_KEY=sk_test_your_key_here
SENDGRID_API_KEY=SG.your_key_here
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=your_secret_here
```

Pre-commit Hook to Block Accidental Commits

```bash
#!/bin/bash
.git/hooks/pre-commit. runs before every commit
Install - cp this file to .git/hooks/pre-commit && chmod +x .git/hooks/pre-commit

Patterns that look like API keys
PATTERNS=(
  'AKIA[0-9A-Z]{16}'                    # AWS access key
  'sk_(live|test)_[0-9a-zA-Z]{24,}'    # Stripe secret key
  'SG\.[a-zA-Z0-9_-]{22}\.[a-zA-Z0-9_-]{43}'  # SendGrid
  'ghp_[a-zA-Z0-9]{36}'                # GitHub PAT
  'xoxb-[0-9]+-[a-zA-Z0-9-]+'         # Slack bot token
  'AIza[0-9A-Za-z_-]{35}'              # Google API key
  'ya29\.[0-9A-Za-z_-]+'              # Google OAuth token
  '[0-9a-f]{32}'                        # Generic 32-char hex (many API keys)
)

Check staged files
for pattern in "${PATTERNS[@]}"; do
  if git diff --cached --unified=0 | grep -qE "^\+" | grep -qE "$pattern" 2>/dev/null; then
    echo "ERROR: Possible API key detected in staged changes."
    echo "Pattern: $pattern"
    echo "Run: git diff --cached to review"
    exit 1
  fi
done

Simpler but effective - check for known key prefixes
if git diff --cached --unified=0 | grep -E "^\+" | grep -qE "(AKIA|sk_live_|sk_test_|SG\.|ghp_|xoxb-|AIza)"; then
  echo "ERROR: Possible API key detected. Aborting commit."
  git diff --cached | grep -E "(AKIA|sk_live_|sk_test_|SG\.|ghp_|xoxb-|AIza)"
  exit 1
fi

exit 0
```

Or use `git-secrets` for broader coverage:

```bash
Install git-secrets
brew install git-secrets  # macOS
Or: git clone https://github.com/awslabs/git-secrets && sudo make install

Register AWS pattern (and custom patterns)
git secrets --register-aws
git secrets --add 'sk_(live|test)_[0-9a-zA-Z]+'
git secrets --add 'SG\.[a-zA-Z0-9_-]{22}'

Install hooks globally
git secrets --install ~/.git-templates/git-secrets
git config --global init.templateDir ~/.git-templates/git-secrets
```

Injecting Keys at Runtime

Local Development with direnv

```bash
Install direnv
brew install direnv  # macOS
sudo apt install direnv  # Linux

.envrc. gitignored, lives per-directory
export STRIPE_SECRET_KEY="sk_test_abc123"
export DATABASE_URL="postgres://localhost/mydb_dev"

Allow the .envrc file for this directory
direnv allow .

Keys are loaded when you cd into the directory
Unloaded when you cd out. no leakage to other shells
```

```bash
.gitignore
.envrc
```

dotenv in Node.js

```javascript
// Load .env only in development. never in production
if (process.env.NODE_ENV !== 'production') {
  require('dotenv').config();
}

// Access keys
const stripeKey = process.env.STRIPE_SECRET_KEY;
if (!stripeKey) {
  throw new Error('STRIPE_SECRET_KEY is required');
}
```

Python

```python
import os
from dotenv import load_dotenv

Only load .env if it exists (development)
load_dotenv()

stripe_key = os.environ.get("STRIPE_SECRET_KEY")
if not stripe_key:
    raise ValueError("STRIPE_SECRET_KEY environment variable is not set")
```

Using AWS Secrets Manager

For production services, keys should come from a managed secret store, not environment variables baked into the deployment.

```python
import boto3
import json
from functools import lru_cache

@lru_cache(maxsize=None)
def get_secret(secret_name: str, region: str = "us-east-1") -> dict:
    """Fetch a secret from AWS Secrets Manager. Cached after first call."""
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

Usage
creds = get_secret("prod/myapp/stripe")
stripe_key = creds["secret_key"]
```

```bash
Create a secret
aws secretsmanager create-secret \
  --name "prod/myapp/stripe" \
  --secret-string '{"secret_key":"sk_live_abc","publishable_key":"pk_live_xyz"}'

Rotate a secret manually
aws secretsmanager rotate-secret --secret-id "prod/myapp/stripe"

Give your EC2/ECS/Lambda access via IAM policy
aws iam put-role-policy \
  --role-name MyAppRole \
  --policy-name SecretsAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/myapp/*"
    }]
  }'
```

Scanning for Leaked Keys in Git History

If a key was ever committed, even for a moment, it should be treated as compromised.

```bash
Scan entire git history with truffleHog
pip install trufflehog
trufflehog git file://. --since-commit HEAD~100

Or with gitleaks (faster)
brew install gitleaks
gitleaks detect --source . --verbose

Check a specific commit range
gitleaks detect --source . --log-opts="HEAD~50..HEAD"
```

If a leak is found:
1. Revoke the key at the provider immediately. do not wait
2. Generate a new key and deploy it
3. Remove the key from history with `git filter-repo` (optional. revocation is the critical step)

```bash
Remove a specific string from all git history
pip install git-filter-repo
git filter-repo --replace-text <(echo "sk_live_abc==>REMOVED_SECRET")
git push --force-with-lease origin main
```

API Key Rotation Automation

Keys with no rotation schedule are keys waiting to be abused after a breach.

```python
#!/usr/bin/env python3
rotate-aws-keys.py. rotates IAM access keys older than 90 days

import boto3
from datetime import datetime, timezone, timedelta

iam = boto3.client("iam")
ROTATION_DAYS = 90

def rotate_old_keys():
    users = iam.list_users()["Users"]
    for user in users:
        username = user["UserName"]
        keys = iam.list_access_keys(UserName=username)["AccessKeyMetadata"]
        for key in keys:
            age = datetime.now(timezone.utc) - key["CreateDate"]
            if age > timedelta(days=ROTATION_DAYS) and key["Status"] == "Active":
                print(f"Rotating key for {username} (age: {age.days} days)")
                # Create new key
                new_key = iam.create_access_key(UserName=username)["AccessKey"]
                print(f"  New key ID: {new_key['AccessKeyId']}")
                # Store new key in Secrets Manager
                # ... (your secret update logic here)
                # Deactivate old key
                iam.update_access_key(
                    UserName=username,
                    AccessKeyId=key["AccessKeyId"],
                    Status="Inactive"
                )
                # Delete after verifying new key works
                # iam.delete_access_key(UserName=username, AccessKeyId=key['AccessKeyId'])

rotate_old_keys()
```

Key Permission Scoping

Create keys with the minimum required permissions.

```bash
Bad - full-access key stored in .env
Good - scoped key that can only read from one S3 bucket

aws iam create-policy \
  --policy-name MyAppS3ReadOnly \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-specific-bucket",
        "arn:aws:s3:::my-specific-bucket/*"
      ]
    }]
  }'
```

Related Articles

- [Privacy-Focused Git Workflow: Preventing Accidental Commits](/privacy-focused-git-workflow-preventing-accidental-commits/)
- [How to Secure Your GitHub Account](/secure-github-account-hardening-guide/)
- [Secure Kubernetes Secrets Management Guide](/kubernetes-secrets-management-guide/)
- [1password Cli Secrets Management Guide](/1password-cli-secrets-management-guide/)
- [Using curl for LinkedIn API](/social-media-data-request-download-guide-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
