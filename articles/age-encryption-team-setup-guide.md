---
layout: default
title: "How to Set Up age Encryption for Teams"
description: "Set up age encryption across a team with shared recipients, key management, and automated pipelines for secure file distribution"
date: 2026-03-22
author: theluckystrike
permalink: /age-encryption-team-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up age Encryption for Teams

[age](https://age-encryption.org) is a simple, modern encryption tool that replaces GPG for most file encryption tasks. For teams it solves the classic problem: encrypt a file so that multiple people can decrypt it, without sharing a single secret key or managing a keyserver.

This guide covers multi-recipient encryption, key distribution workflows, and automating age in CI/CD pipelines.

## Key Takeaways

- **Only share the `age1...`**: public key line.
- Useful for long-term archival keys.
- **This guide covers prerequisites**: key generation for each team member, creating a team recipients file, with specific setup instructions
- **Setup and configuration**: Step-by-step instructions included for each tool discussed

## Prerequisites

- age 1.1+ installed on all machines
- A way to distribute public keys (git repo, internal wiki, or age-plugin)

```bash
# Install age on Linux
curl -L https://github.com/FiloSottile/age/releases/download/v1.1.1/age-v1.1.1-linux-amd64.tar.gz \
  | tar xz && sudo mv age/age age/age-keygen /usr/local/bin/

# macOS
brew install age

# Verify
age --version
```

### Step 1: Generate Keys for Each Team Member

Every team member generates their own key pair and shares only the public key.

```bash
# Generate a key pair — store the private key safely (password manager or ~/.age/)
age-keygen -o ~/.age/identity.txt

# The output looks like:
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
# Secret key is in the file

# Extract just the public key to share
age-keygen -y ~/.age/identity.txt
# age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

Never share `identity.txt`. Only share the `age1...` public key line.

### Step 2: Create a Team Recipients File

Store all team public keys in a single file checked into your repo.

```
# .age-recipients — commit this file to your repository
# Format: one key per line, comments with #

# Engineering
age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p  # alice
age1xmss5fpdz5fthd9gy9l2yrjq3t9qfklkzs8gp8cv5g4htj2vj7skj3ll5  # bob
age17g0rl4kxvz9qeqjdl55xfks4fk3xqzm3czl6khh7afdh6plst3zs4zxhay  # carol

# DevOps
age1eky0gn4aq9p8pqxxc9njk3zjlrx0n3efc2j6czs0k0yjvplpzr3q5hxzg5  # dave
```

```bash
# Encrypt a secrets file to all team members
age -R .age-recipients secrets.env -o secrets.env.age

# Decrypt (any team member with their identity.txt can do this)
age -d -i ~/.age/identity.txt secrets.env.age > secrets.env
```

### Step 3: Encrypt to Subsets — Roles and Access Tiers

For role-based access, maintain separate recipient files.

```bash
mkdir -p .age-keys
cat > .age-keys/devops.txt << 'EOF'
age1eky0gn4aq9p8pqxxc9njk3zjlrx0n3efc2j6czs0k0yjvplpzr3q5hxzg5  # dave
age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p  # alice (lead)
EOF

cat > .age-keys/all-engineers.txt << 'EOF'
age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p  # alice
age1xmss5fpdz5fthd9gy9l2yrjq3t9qfklkzs8gp8cv5g4htj2vj7skj3ll5  # bob
age17g0rl4kxvz9qeqjdl55xfks4fk3xqzm3czl6khh7afdh6plst3zs4zxhay  # carol
age1eky0gn4aq9p8pqxxc9njk3zjlrx0n3efc2j6czs0k0yjvplpzr3q5hxzg5  # dave
EOF

# Encrypt production database creds — only devops can decrypt
age -R .age-keys/devops.txt db-prod.env -o secrets/db-prod.env.age

# Encrypt dev API keys — all engineers can decrypt
age -R .age-keys/all-engineers.txt dev-keys.env -o secrets/dev-keys.env.age
```

### Step 4: Store Encrypted Secrets in Git

A common pattern is to keep encrypted files in the repo and plaintext in `.gitignore`.

```bash
# .gitignore
*.env
*.key
*.pem
!*.env.age   # allow encrypted versions

# Makefile targets for the team
decrypt-dev:
	age -d -i ~/.age/identity.txt secrets/dev-keys.env.age > dev-keys.env

decrypt-prod:
	age -d -i ~/.age/identity.txt secrets/db-prod.env.age > db-prod.env

encrypt-dev:
	age -R .age-keys/all-engineers.txt dev-keys.env -o secrets/dev-keys.env.age

encrypt-prod:
	age -R .age-keys/devops.txt db-prod.env -o secrets/db-prod.env.age
```

### Step 5: Configure CI/CD Integration with GitHub Actions

For CI pipelines, store the private key as a base64-encoded secret.

```bash
# Encode your CI key for storage in GitHub Secrets
base64 -w0 ~/.age/ci-identity.txt
# Paste the output as CI_AGE_KEY in GitHub repo secrets
```

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install age
        run: |
          curl -L https://github.com/FiloSottile/age/releases/download/v1.1.1/age-v1.1.1-linux-amd64.tar.gz \
            | tar xz && sudo mv age/age age/age-keygen /usr/local/bin/

      - name: Decrypt secrets
        env:
          CI_AGE_KEY: ${{ secrets.CI_AGE_KEY }}
        run: |
          echo "$CI_AGE_KEY" | base64 -d > /tmp/ci-identity.txt
          age -d -i /tmp/ci-identity.txt secrets/db-prod.env.age > db-prod.env
          rm /tmp/ci-identity.txt

      - name: Deploy
        run: ./deploy.sh
```

The CI key is a separate key pair added to `.age-keys/devops.txt`. When a team member leaves, remove their key from the recipients file, re-encrypt all secrets with `age -R`, and commit.

### Step 6: Rotate Keys

When a team member leaves or a key is compromised:

```bash
#!/bin/bash
# rotate-secrets.sh — re-encrypt all .age files with updated recipients

SECRETS_DIR="secrets"
RECIPIENTS_DIR=".age-keys"

for age_file in "$SECRETS_DIR"/*.age; do
  base="${age_file%.age}"
  name=$(basename "$base")

  # Determine which recipients file applies
  if [[ "$name" == *"prod"* ]]; then
    recipients="$RECIPIENTS_DIR/devops.txt"
  else
    recipients="$RECIPIENTS_DIR/all-engineers.txt"
  fi

  # Decrypt with your own key, re-encrypt with new recipients
  plaintext=$(age -d -i ~/.age/identity.txt "$age_file")
  echo "$plaintext" | age -R "$recipients" -o "${age_file}.new"
  mv "${age_file}.new" "$age_file"
  echo "Rotated: $age_file"
done
```

```bash
chmod +x rotate-secrets.sh
# After removing the departed member's key from the recipients files:
./rotate-secrets.sh
git add secrets/
git commit -m "security: rotate secrets after key removal"
```

### Step 7: Use age with SSH Keys

If your team already uses SSH keys, age can derive recipients from them — no separate key generation needed.

```bash
# Encrypt to someone's GitHub SSH key
curl https://github.com/alice.keys | age -R - secrets.env -o secrets.env.age

# Or use their key directly
age -r "$(curl -s https://github.com/alice.keys | head -1)" secrets.env -o secrets.env.age
```

This works with Ed25519 and RSA SSH keys. The recipient decrypts with their SSH private key:

```bash
# Decrypt using SSH identity
age -d -i ~/.ssh/id_ed25519 secrets.env.age > secrets.env
```

### Step 8: Set Up Passphrase -Protected Identities

For extra protection on the private key file, use a passphrase. Useful for long-term archival keys.

```bash
# Generate a passphrase-protected identity
age-keygen | age -p > ~/.age/identity-protected.txt.age

# Decrypt the identity before use
age -d ~/.age/identity-protected.txt.age > /tmp/identity.txt
age -d -i /tmp/identity.txt archive.tar.age > archive.tar
shred -u /tmp/identity.txt
```

### Step 9: Audit Who Has Access

Since all public keys are in version-controlled files, access auditing is straightforward.

```bash
# Who can decrypt prod secrets?
cat .age-keys/devops.txt

# Check git history for access changes
git log --oneline -- .age-keys/devops.txt

# List all secrets and their recipient files
for f in secrets/*.age; do
  echo "=== $f ==="
  # age does not expose recipients, but the Makefile/script defines this
done
```

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [How to Use age Encryption for Secure File Sharing](/privacy-tools-guide/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
- [Encrypted File Sync for Teams Comparison](/privacy-tools-guide/encrypted-file-sync-for-teams-comparison/)
- [Secure API Key Management for Developers](/privacy-tools-guide/secure-api-key-management-developers/)
---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
