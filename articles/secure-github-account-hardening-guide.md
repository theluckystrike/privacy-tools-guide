---
---
layout: default
title: "How to Secure Your GitHub Account"
description: "Harden your GitHub account with 2FA, SSH key management, token auditing, branch protection, and secret scanning to prevent account takeovers"
date: 2026-03-22
author: theluckystrike
permalink: /secure-github-account-hardening-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Secure Your GitHub Account

GitHub accounts hold source code, credentials, deployment keys, and CI/CD pipelines. A compromised account is a supply chain attack against everyone who uses your code. This guide covers the full hardening checklist.

## Step 1: Enable Hardware-Key 2FA

GitHub supports FIDO2/WebAuthn security keys (YubiKey, Google Titan). This is strictly better than TOTP apps, which are phishable.

1. Go to **Settings → Password and authentication → Two-factor authentication**
2. Click **Add security key**
3. Tap your physical key when prompted
4. Store your 2FA recovery codes in your password manager, not a text file

```bash
# Verify 2FA is active via GitHub CLI
gh auth status
# Should show: ✓ Logged in to github.com as yourname (2FA: enabled)

# List your active sessions
gh api user/sessions --jq '.[].created_at'
```

If you cannot use a hardware key, use an authenticator app (Aegis on Android, Raivo on iOS). Never use SMS 2FA — it is vulnerable to SIM swapping.

## Step 2: Audit and Rotate SSH Keys

```bash
# List all SSH keys on your account
gh api user/keys --jq '.[] | "\(.id) \(.title) created:\(.created_at)"'

# Remove a key you no longer recognize
gh api -X DELETE user/keys/{key_id}

# Generate a new Ed25519 key for GitHub
ssh-keygen -t ed25519 -C "github-$(hostname)-$(date +%Y%m)" -f ~/.ssh/github_ed25519

# Add to ssh-agent
ssh-add ~/.ssh/github_ed25519

# Add the public key to GitHub
gh ssh-key add ~/.ssh/github_ed25519.pub --title "laptop-$(date +%Y%m)"

# Use a dedicated config entry
cat >> ~/.ssh/config << 'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes
EOF

# Test
ssh -T git@github.com
```

Rotate keys at least annually. If you ever share a machine with someone else, rotate immediately.

## Step 3: Audit Personal Access Tokens

PATs are the most common attack vector — developers create them, forget them, and they live forever.

```bash
# List all fine-grained tokens (GitHub CLI)
gh api /user/personal-access-tokens --jq '.[] | "\(.id) \(.name) expires:\(.expires_at) last_used:\(.last_used_at)"'

# List classic tokens (no API — must be done in UI)
# Settings → Developer settings → Personal access tokens → Tokens (classic)
```

Rules for PATs:
- Set expiration on every token (max 1 year, prefer 90 days)
- Use fine-grained tokens scoped to specific repos
- Never create tokens with `repo` full scope when `contents:read` suffices
- Store tokens in a password manager, not shell history or `.bashrc`

```bash
# Create a minimal fine-grained PAT for a specific repo via CLI
gh auth token  # shows current token
gh auth refresh --scopes repo  # refresh with explicit scopes only if needed

# Audit your shell history for leaked tokens
grep -E 'ghp_|github_pat_' ~/.bash_history ~/.zsh_history 2>/dev/null
# If found: rotate the token immediately, then clear history
```

## Step 4: Enable Secret Scanning and Push Protection

Secret scanning detects committed credentials in public and private repos.

```bash
# Enable for all repos via API (run for each org you own)
gh api -X PATCH /repos/{owner}/{repo} \
  --field security_and_analysis.secret_scanning.status=enabled \
  --field security_and_analysis.secret_scanning_push_protection.status=enabled

# Check if secret scanning is enabled
gh api /repos/{owner}/{repo} --jq '.security_and_analysis'
```

For organizations:

```bash
# Enable secret scanning org-wide
gh api -X PATCH /orgs/{org} \
  --field security_and_analysis.secret_scanning.status=enabled

# List detected secrets across an org
gh api /orgs/{org}/secret-scanning/alerts --jq '.[] | "\(.secret_type) \(.state) \(.html_url)"'
```

## Step 5: Configure Branch Protection Rules

Protect your main branch from force pushes and direct commits.

```bash
# Set branch protection via API
gh api -X PUT /repos/{owner}/{repo}/branches/main/protection \
  --input - << 'EOF'
{
  "required_status_checks": {
    "strict": true,
    "contexts": []
  },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true
  },
  "restrictions": null,
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false
}
EOF

# Verify
gh api /repos/{owner}/{repo}/branches/main/protection --jq '{
  force_push_blocked: .allow_force_pushes.enabled,
  require_pr: .required_pull_request_reviews.required_approving_review_count
}'
```

## Step 6: Audit OAuth App Access

Third-party apps that have OAuth access to your GitHub account can read private repos.

```bash
# List authorized OAuth apps (API only returns installed apps, UI shows OAuth)
# Settings → Applications → Authorized OAuth Apps

# Via API — list GitHub Apps installed on your account
gh api /user/installations --jq '.installations[] | "\(.id) \(.app_slug) \(.repository_selection)"'

# Revoke a specific app installation
gh api -X DELETE /user/installations/{installation_id}
```

Revoke access for any app you no longer actively use.

## Step 7: Harden Actions Permissions

GitHub Actions workflows that run on pull requests from forks can exfiltrate secrets if misconfigured.

```bash
# Restrict Actions permissions for a repo
gh api -X PUT /repos/{owner}/{repo}/actions/permissions \
  --field enabled=true \
  --field allowed_actions=selected

# Allow only verified actions + your own
gh api -X PUT /repos/{owner}/{repo}/actions/permissions/selected-actions \
  --field github_owned_allowed=true \
  --field verified_allowed=true \
  --field patterns_allowed='["your-org/*"]'

# Disable pull-request triggers from forks writing to secrets
gh api -X PUT /repos/{owner}/{repo}/actions/permissions/workflow \
  --field default_workflow_permissions=read \
  --field can_approve_pull_request_reviews=false
```

Also audit your workflow files for `pull_request_target` with checkout of the fork's code — that pattern is frequently exploited.

```bash
# Find potentially dangerous workflow patterns
grep -r "pull_request_target" /path/to/repo/.github/workflows/
grep -r "ref: \${{ github.event.pull_request.head.ref }}" /path/to/repo/.github/workflows/
```

## Step 8: Configure Vigilant Mode for Commit Signing

Vigilant mode marks any unsigned commit as "unverified" on your profile — useful for detecting spoofed commits.

1. **Settings → SSH and GPG keys → Vigilant mode → Enable**

```bash
# Sign commits with SSH key (simpler than GPG)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/github_ed25519.pub
git config --global commit.gpgsign true

# Add your SSH signing key to GitHub
# Settings → SSH and GPG keys → New SSH key → Key type: Signing Key

# Test a signed commit
git commit --allow-empty -m "test: verify commit signing"
git log --show-signature -1
```

## Step 9: Regular Security Audit Script

```bash
#!/bin/bash
# github-audit.sh — run monthly

echo "=== SSH Keys ==="
gh api user/keys --jq '.[] | "\(.id) \(.title) added:\(.created_at)"'

echo ""
echo "=== Signed-in Sessions ==="
gh api user/sessions 2>/dev/null | jq -r '.[].created_at' || echo "Not available via API — check UI"

echo ""
echo "=== Recent Security Events ==="
gh api /users/"$(gh api user --jq .login)"/events/public \
  --jq '.[] | select(.type == "CreateEvent" or .type == "DeleteEvent") | "\(.created_at) \(.type) \(.repo.name)"' \
  | head -20

echo ""
echo "=== Active PATs (fine-grained) ==="
gh api /user/personal-access-tokens --jq '.[] | "\(.name) expires:\(.expires_at)"' 2>/dev/null \
  || echo "Use UI for classic tokens: github.com/settings/tokens"
```

## Related Articles

- [How to Use YubiKey for SSH Authentication](/privacy-tools-guide/articles/how-to-use-yubikey-for-ssh-authentication-guide/)
- [How to Harden SSH Server Configuration](/privacy-tools-guide/how-to-harden-ssh-server-configuration/)
- [How To Create Anonymous GitHub Account For Open Source](/privacy-tools-guide/how-to-create-anonymous-github-account-for-open-source-contr/)
- [Secure Shell Hardening Beyond SSH Config](/privacy-tools-guide/secure-shell-hardening-beyond-ssh-config/)
- [Secure API Key Management for Developers](/privacy-tools-guide/secure-api-key-management-developers/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
