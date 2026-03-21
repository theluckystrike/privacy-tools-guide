---
layout: default
title: "Secure Password Sharing for Teams"
description: "How to share credentials securely across teams. Covers Bitwarden Organizations, Passbolt, 1Password Teams, and one-time secret links for guests."
date: 2026-03-21
author: theluckystrike
permalink: secure-password-sharing-teams-guide
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Most teams share passwords via Slack messages, shared spreadsheets, or email threads — all of which are logged, searchable, and often breach-indexed. Proper credential sharing means every access is auditable, every credential is encrypted in transit, and departures don't require a password rotation circus.

This guide covers tools and workflows for teams ranging from 2 people to 50.

## Why This Matters

When someone leaves a company and you haven't tracked which credentials they had access to:
- You don't know what to rotate
- Old credentials may still be in their personal password manager
- You have no audit trail to show in a compliance review

Proper shared credential management solves all three.

## The Core Requirements

A secure team password sharing setup should provide:

1. **Encryption in transit and at rest** — credentials encrypted before leaving the device
2. **Role-based access** — not everyone needs every password
3. **Audit log** — who accessed what credential and when
4. **Offboarding workflow** — remove access without breaking anything
5. **Secret rotation** — process for changing credentials when someone leaves
6. **No plaintext over chat** — zero credentials sent via Slack, email, or SMS

## Option 1: Bitwarden Organizations

Bitwarden's free organization tier supports up to two users. The Teams plan (~$3/user/month) supports unlimited users with advanced sharing and audit logs.

### Setting Up an Organization

1. Create an organization at `https://bitwarden.com` or your self-hosted Vaultwarden instance
2. Go to **Organizations > Settings > Members > Invite Member**
3. Assign roles: Owner, Admin, Manager, or User

### Collections (Folders With Access Control)

Collections group credentials by team or purpose:

```
Organization
├── Engineering
│   ├── AWS Production
│   ├── GitHub Org tokens
│   └── Database credentials
├── Marketing
│   ├── Social media accounts
│   └── Analytics tools
└── Shared (all-team)
    ├── Company WiFi
    └── Office alarm code
```

Create a collection and assign which groups can access it:

- **Read** — can view and copy credentials
- **Read + hide passwords** — can autofill but not see the raw password
- **Read/Write** — can also edit
- **Manager** — can modify the collection membership

### Bitwarden CLI for Automation

```bash
# Install Bitwarden CLI
npm install -g @bitwarden/cli

# Login
bw login

# List collections in an organization
bw list org-collections --organizationid YOUR_ORG_ID

# Create an item in a specific collection
bw get template item | jq '.name="AWS Production Key" | .login.username="deploy" | .login.password="'"$GENERATED_PASSWORD"'"' | \
  bw encode | bw create item --collectionids COLLECTION_ID
```

## Option 2: 1Password Teams

1Password Teams includes travel mode, item history (restore previous passwords), and document storage alongside credentials.

### Team Vaults

```
Team Account
├── Shared vault (all team members)
├── Engineering vault (engineers + leads)
├── HR vault (HR + CEO)
└── Personal vault (individual, private)
```

Vault permissions are set at the group level. Create groups matching your org chart, assign people to groups, assign groups to vaults.

### 1Password CLI for CI/CD

1Password's `op` CLI is particularly useful for injecting secrets into CI pipelines without storing them in environment variables:

```bash
# Install op CLI
brew install 1password-cli   # macOS
# Or download from https://1password.com/downloads/command-line/

# Sign in
op signin

# Inject secrets into a command
op run --env-file=".env.1p" -- docker-compose up

# .env.1p references vault items
# DB_PASSWORD=op://Engineering/PostgreSQL Production/password
# AWS_ACCESS_KEY=op://Engineering/AWS Deploy/access_key_id
```

This pattern means secrets are never written to `.env` files in the repo.

## Option 3: Passbolt (Open Source, Self-Hosted)

Passbolt uses OpenPGP — each secret is encrypted with the public keys of each authorized user. Not even the server admin can read passwords.

### Sharing a Password with a Team Member

```bash
# Install passbolt CLI
npm install -g passbolt-cli

# Configure
passbolt config set --server https://passbolt.yourdomain.com \
  --fingerprint YOUR_KEY_FINGERPRINT \
  --private-key path/to/private.key

# Share a resource with another user
passbolt share --resource-id RESOURCE_UUID \
  --user-fingerprint RECIPIENT_KEY_FINGERPRINT \
  --permission READ
```

### Passbolt Groups

Create a group for each team:
- Settings > Groups > Create group
- Add members and assign group permissions to folders
- Sharing a folder with a group shares all contained passwords

When someone leaves:
1. Remove them from relevant groups
2. Rotate any credentials they had access to (Passbolt's activity log shows exactly which ones)
3. Deactivate their account

## Option 4: One-Time Secret Links for Guests

For sharing a credential with a contractor, vendor, or client — someone who isn't on your team's password manager:

### Self-Hosted with Yopass

```bash
docker run -d -p 1337:1337 --name yopass \
  -e MEMCACHED=memcached:11211 \
  --link memcached \
  jhaals/yopass

docker run -d --name memcached memcached:1.6
```

Access at `http://localhost:1337`. The secret is encrypted with a key included in the share URL. Once read (or after a set time), the encrypted value is deleted from the backend.

### Using Bitwarden Send

Bitwarden has a built-in one-time share feature:

```bash
# Create a Send via CLI
bw send create \
  --text "Temporary API key: abc123xyz" \
  --deletionDate "2026-03-28T00:00:00Z" \
  --maxAccessCount 1
```

Or via the web UI: **Send > New Send > Text** — set expiration and access count, share the link.

### Using curl Against a Public API (Quick Option)

For internal use when self-hosting isn't set up yet:

```bash
# Create a secret on onetimesecret.com
curl -X POST https://onetimesecret.com/api/v1/share \
  -u 'your@email.com:API_TOKEN' \
  -d 'secret=temporary-password-123&ttl=3600'
```

This is a hosted service — only use for non-critical, temporary credentials.

## Access Audit Workflow

Run this process when anyone leaves the organization:

```bash
# For Bitwarden via CLI — list events for a specific user
bw list --organizationid ORG_ID \
  | jq '.[] | select(.userId == "USER_UUID")'
```

In the Bitwarden web vault: Admin Console > Reporting > Event Logs > filter by user.

Steps for offboarding:
1. Immediately revoke organization access (don't wait)
2. Pull the event log for the last 90 days filtered to their user ID
3. Identify every credential they accessed in the last 30 days
4. Rotate those credentials in priority order (admin credentials first)
5. Update any systems using rotated credentials
6. Document the rotation in your change log

## Team Password Hygiene Rules

Write these into a team security policy document:

- No credentials in Slack, Teams, or email — ever
- New credentials go into the password manager before being used anywhere
- Shared credentials use generated passwords, not memorable ones
- When someone leaves, rotation happens within 48 hours for high-privilege credentials
- Service accounts use separate credentials from human accounts
- Production credentials are never on developer laptops (use a secrets manager for CI/CD)

## Related Reading

- [Privacy-Focused Password Sharing for Families](/privacy-focused-password-sharing-for-families-guide)
- [How to Manage Team Password Vault Permissions](/how-to-manage-team-password-vault-permissions-across-enterpr)
- [Self-Hosted Password Manager Comparison](/self-hosted-password-manager-comparison)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
