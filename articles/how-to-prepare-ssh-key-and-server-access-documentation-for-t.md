---
layout: default
title: "How To Prepare Ssh Key And Server Access Documentation For T"
description: "A practical guide for developers and power users on documenting SSH keys, server credentials, and access details for technical estate inheritance."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-prepare-ssh-key-and-server-access-documentation-for-t/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Technical estate inheritance refers to the process of transferring digital infrastructure access from one person or team to another. Whether you are a sysadmin preparing for retirement, a lead developer transitioning projects, or an organization ensuring business continuity, proper documentation of SSH keys and server access is critical. Without it, your team faces unnecessary downtime, security risks, or costly recovery efforts.

This guide walks through creating SSH and server access documentation that makes technical inheritance and secure.

## Why Technical Estate Documentation Matters

Every organization has critical infrastructure that only certain individuals can access. When those individuals become unavailable—whether due to role changes, extended leave, or unexpected circumstances—their technical knowledge dies with them. This creates operational risk.

Good documentation serves multiple purposes. It enables continuity during personnel changes. It supports compliance requirements for access auditing. It reduces the time required to onboard new technical staff. And it provides a clear inventory of what exists in your technical environment.

## Inventorying SSH Keys and Access Points

Before you can document anything, you need to know what exists. Start by creating a complete inventory of every system that requires SSH access.

### Discovering Your SSH Keys

Run this command to list all SSH keys in your default location:

```bash
ls -la ~/.ssh/
```

For organizations with multiple keys across different projects, you may need to search more broadly:

```bash
find /home -name "id_*" -type f 2>/dev/null
find /root -name "id_*" -type f 2>/dev/null
```

Document each key with the following information:

- Key filename and full path
- Key type (RSA, Ed25519, ECDSA)
- Creation date (if discoverable)
- Associated servers or services
- Passphrase status (set or unset)
- Key purpose or project association

### Mapping Server Access

For each server or service, document the connection details:

```bash
# Example inventory entry format
server:
  hostname: prod-web-01.example.com
  ip: 10.0.1.25
  ssh_port: 22
  user: deploy
  key_path: /home/username/.ssh/prod_deploy_key
  purpose: Production web server
  owner: Platform Team
  last_accessed: 2026-03-01
```

Create a master spreadsheet or YAML file that tracks all servers, their purpose, who has access, and the authentication method used.

## Documenting Authentication Methods

SSH access typically relies on several authentication mechanisms. Document each one thoroughly.

### Key-Based Authentication

For key-based access, store the public key fingerprint rather than the private key itself:

```bash
ssh-keygen -lf ~/.ssh/your_key.pub
```

This outputs something like `256 SHA256:abcd1234... (ED25519)`. Record this fingerprint. It allows verification without exposing credentials.

Create a structured document that maps each server to its authorized keys:

```yaml
authorized_keys:
  prod-web-01:
    - key_fingerprint: "SHA256:abc123..."
      added_date: 2025-06-15
      added_by: john.doe
      purpose: "Deployment automation"
    - key_fingerprint: "SHA256:def456..."
      added_date: 2025-08-20
      added_by: jane.smith
      purpose: "Emergency access"
```

### Password and MFA Documentation

If any systems use password authentication (which we recommend avoiding), document the password vault location and access procedures. For systems with MFA, document which MFA method is configured:

- TOTP (Time-based One-Time Password)
- Hardware security key (YubiKey, SoloKey)
- Push notification to mobile app

Never store actual passwords in plain text. Instead, reference your password manager and document the vault structure.

## Creating Access Runbooks

Documentation is only useful if others can understand it. Create clear runbooks for common access scenarios.

### Basic SSH Connection

For each server, provide the exact connection command:

```bash
ssh -i ~/.ssh/prod_deploy_key deploy@prod-web-01.example.com
```

If non-standard ports are used, include that information:

```bash
ssh -i ~/.ssh/prod_deploy_key -p 2222 deploy@prod-web-01.example.com
```

### Jump Host and Bastion Access

Many organizations use bastion hosts or jump servers. Document the complete chain:

```bash
# First hop: bastion
ssh -i ~/.ssh/bastion_key admin@bastion.example.com

# Second hop: internal server (from bastion)
ssh -i ~/.ssh/internal_key webserver@10.0.1.25
```

For SSH config convenience, document the SSH config entries:

```bash
# Add to ~/.ssh/config
Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/bastion_key
    ForwardAgent yes

Host prod-web-01
    HostName 10.0.1.25
    User webserver
    IdentityFile ~/.ssh/internal_key
    ProxyJump bastion
```

### Emergency Access Procedures

Document what to do when normal access fails. Include:

- Steps for password reset
- Contact information for infrastructure team
- Procedure for requesting new SSH keys
- Recovery options if SSH config is broken

## Secure Storage and Access Control

Your documentation itself becomes a high-value target. Protect it accordingly.

### Encryption at Rest

Store sensitive documentation in an encrypted volume or password-protected vault. For plaintext documentation, encrypt the files:

```bash
# Using age (recommended)
age -p -o documentation.age README.md

# Using GPG
gpg -c documentation.md
```

### Access Restrictions

Limit who can view the documentation. Use access control lists or group-based permissions. Audit access regularly.

### Regular Updates

Set a schedule to review and update documentation. We recommend quarterly reviews for active projects and annual reviews for stable infrastructure. Include a last-updated timestamp in your documents:

```yaml
documentation_meta:
  version: "1.2"
  last_updated: "2026-03-10"
  next_review: "2026-06-10"
  maintained_by: platform-team@example.com
```

## Transition Checklist

When actual handover occurs, use this checklist to ensure nothing is missed:

- [ ] All SSH keys transferred and tested
- [ ] MFA devices or backup codes handed over
- [ ] Password manager access confirmed
- [ ] SSH config files reviewed and updated
- [ ] Jump host access verified
- [ ] All servers accessible from new keys
- [ ] Documentation reviewed for accuracy
- [ ] Emergency contacts updated
- [ ] Previous access revoked (where applicable)
- [ ] Sign-off from both parties

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
