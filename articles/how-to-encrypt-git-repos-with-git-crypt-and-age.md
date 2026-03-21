---
layout: default
title: "How to Encrypt Git Repos with git-crypt and age"
description: "Full guide to encrypting sensitive files in Git repos using git-crypt and age. Team key management, CI/CD integration, and GPG-free workflows covered"
date: 2026-03-20
author: theluckystrike
permalink: /how-to-encrypt-git-repos-with-git-crypt-and-age/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

git-crypt transparently encrypts files in a Git repository -- they are encrypted at rest and in the remote, but automatically decrypted when you check them out on an unlocked machine. age handles the actual file encryption for workflows where git-crypt is not suitable: secret shares, offline archives, or encrypting entire directories before pushing. Use git-crypt for secrets that live alongside code (API keys, certs, .env files). Use age directly when you need to encrypt artifacts, share secrets outside the repo, or avoid GPG entirely.

## Install git-crypt

```bash
# macOS
brew install git-crypt

# Ubuntu/Debian
sudo apt install git-crypt

# Verify
git-crypt --version
```

## Initialize git-crypt in a Repository

```bash
cd /path/to/your-repo

# Initialize -- creates .git-crypt/ directory
git-crypt init

# Export the symmetric key (back this up -- losing it means losing access)
git-crypt export-key ~/backup/git-crypt-key.bin
chmod 600 ~/backup/git-crypt-key.bin
```

## Configure Which Files Get Encrypted

Edit or create .gitattributes in the repo root:

```
# .gitattributes

# Encrypt all .env files anywhere in the repo
**/.env filter=git-crypt diff=git-crypt

# Encrypt specific secret files
secrets/api-keys.json filter=git-crypt diff=git-crypt
config/production.yml filter=git-crypt diff=git-crypt

# Encrypt entire directory
secrets/** filter=git-crypt diff=git-crypt

# Encrypt .pem and .key files everywhere
*.pem filter=git-crypt diff=git-crypt
*.key filter=git-crypt diff=git-crypt
```

After adding the attributes, stage and commit the files:

```bash
git add .gitattributes secrets/api-keys.json
git commit -m "Add git-crypt encryption config"
git push

# Verify files are encrypted in the remote
git show HEAD:secrets/api-keys.json  # Shows encrypted binary
git-crypt status                      # Shows which files are encrypted
```

## Add Team Members via GPG Keys

git-crypt multi-user model uses GPG: each team member GPG public key can unlock the repo.

```bash
# Each team member: generate or export their GPG public key
gpg --gen-key
gpg --export --armor alice@example.com > alice.pub

# Repo admin: import the key and add them
gpg --import alice.pub
git-crypt add-gpg-user alice@example.com

# Commit the key blob
git add .git-crypt/
git commit -m "Add alice as git-crypt collaborator"
git push
```

Alice then unlocks on her machine:

```bash
git clone git@github.com:org/repo.git
cd repo
git-crypt unlock  # Uses her GPG private key automatically
cat secrets/api-keys.json  # Now readable
```

## GPG-Free Workflow: Use a Shared Key File

For teams that want to avoid GPG complexity, share the symmetric key via a secrets manager:

```bash
# Repo admin: export key and store in 1Password / Vault / AWS Secrets Manager
git-crypt export-key - | base64 > /tmp/git-crypt-key.b64
# Store this base64 string in your secrets manager

# Team member: retrieve and unlock
your-secrets-tool get git-crypt-key | base64 -d > /tmp/git-crypt-key.bin
git-crypt unlock /tmp/git-crypt-key.bin
shred -u /tmp/git-crypt-key.bin
```

## CI/CD Integration

Store the git-crypt key as a base64 CI secret and unlock during the pipeline:

```yaml
# GitHub Actions
name: Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install git-crypt
        run: sudo apt-get install -y git-crypt

      - name: Unlock secrets
        env:
          GIT_CRYPT_KEY: ${{ secrets.GIT_CRYPT_KEY }}
        run: |
          echo "$GIT_CRYPT_KEY" | base64 -d > /tmp/git-crypt-key.bin
          git-crypt unlock /tmp/git-crypt-key.bin
          shred -u /tmp/git-crypt-key.bin

      - name: Deploy
        run: ./deploy.sh
```

For GitLab CI:

```yaml
unlock_and_deploy:
  script:
    - apt-get install -y git-crypt
    - echo "$GIT_CRYPT_KEY" | base64 -d > /tmp/gckey.bin
    - git-crypt unlock /tmp/gckey.bin
    - shred -u /tmp/gckey.bin
    - ./deploy.sh
```

## Encrypt Individual Files with age

For files you want to keep out of the repo entirely, encrypt them with age before committing:

```bash
# Install age
brew install age  # macOS
# or: go install filippo.io/age/cmd/age@latest

# Generate a key pair
age-keygen -o ~/.age/key.txt
# Public key printed to stdout: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Encrypt a secrets file
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
    -o secrets/prod-env.age \
    secrets/prod-env.txt

# Add plaintext to .gitignore, commit only the encrypted version
echo "secrets/prod-env.txt" >> .gitignore
git add secrets/prod-env.age .gitignore
git commit -m "Add encrypted production secrets"

# Decrypt
age -d -i ~/.age/key.txt -o secrets/prod-env.txt secrets/prod-env.age
```

## Multi-Recipient age Encryption for Teams

Encrypt to multiple recipients -- anyone with their private key can decrypt:

```bash
# Collect team members public keys
# alice: age1abc123...
# bob:   age1def456...
# ci:    age1ghi789...

age -r age1abc123... \
    -r age1def456... \
    -r age1ghi789... \
    -o secrets/prod-env.age \
    secrets/prod-env.txt

# Decrypt (any recipient's key works)
age -d -i ~/.age/key.txt secrets/prod-env.age
```

Store all team public keys in a file for repeatability:

```bash
# recipients.txt
# age1abc123...  alice
# age1def456...  bob
# age1ghi789...  ci-runner

# Encrypt to all recipients at once
age $(grep -v '^#' recipients.txt | sed 's/^/-r /') \
    -o secrets/prod-env.age secrets/prod-env.txt
```

## Audit and Rotate Keys

```bash
# List all git-crypt encrypted files
git-crypt status -e | head -20

# For age: stop including the departing user's key in future encryptions
# Re-encrypt existing files with new recipient list
age -r age1alice... -r age1bob... \
    -o secrets/prod-env.age \
    <(age -d -i ~/.age/key.txt secrets/prod-env.age.bak)

# Rotate git-crypt key: remove user's key blob and reinitialize
rm -rf .git-crypt/keys/default/0/<removed-user-key-id>/
git-crypt init
# All remaining users must re-add themselves
```

## Preventing Accidental Plaintext Commits

The biggest failure mode for git-crypt is committing a secret file before adding the filter attribute to `.gitattributes`. Once the plaintext is in history, you need a history rewrite — expensive and disruptive. Add a pre-commit hook to catch this before it happens:

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit — block commits of known secret files if git-crypt is not unlocked

# Check if git-crypt is initialized
if [ ! -d ".git-crypt" ]; then
  exit 0  # No git-crypt, nothing to check
fi

# Verify that encrypted files are listed in .gitattributes
STAGED=$(git diff --cached --name-only)

for file in $STAGED; do
  # Check if the file matches a git-crypt pattern
  if git check-attr filter "$file" | grep -q "git-crypt"; then
    # Verify the file is actually being stored encrypted
    if git-crypt status "$file" 2>/dev/null | grep -q "not encrypted"; then
      echo "ERROR: $file is staged but not encrypted by git-crypt."
      echo "Run: git-crypt unlock"
      exit 1
    fi
  fi
done
```

Install the hook in every clone with a setup script:

```bash
#!/usr/bin/env bash
# scripts/setup-hooks.sh
cp scripts/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
echo "Git hooks installed."
```

Add `./scripts/setup-hooks.sh` to your onboarding README so new team members run it on first clone.

## Migrating Existing Secrets into git-crypt

If secrets are already in your repository history in plaintext, you need to purge them before adding encryption. The safest approach:

```bash
# Step 1: Rotate the actual secrets first — assume they are compromised
# (change API keys, rotate certs, generate new passwords)

# Step 2: Remove the file from history using git-filter-repo
pip install git-filter-repo
git filter-repo --path secrets/api-keys.json --invert-paths

# Step 3: Add git-crypt attributes
echo 'secrets/api-keys.json filter=git-crypt diff=git-crypt' >> .gitattributes
git-crypt init
git-crypt export-key ~/backup/git-crypt-key.bin

# Step 4: Add the new (rotated) secret file and commit
git add .gitattributes secrets/api-keys.json
git commit -m "Encrypt api-keys.json with git-crypt"

# Step 5: Force push the rewritten history
git push --force-with-lease origin main
```

Alert all collaborators to re-clone after a history rewrite. Any stale clones can re-introduce the purged files if pushed again.

## Using age-plugin-yubikey for Hardware Key Protection

For higher-assurance environments, the `age-plugin-yubikey` plugin lets you store age private keys on a YubiKey rather than on disk:

```bash
# Install the plugin
cargo install age-plugin-yubikey

# Initialize a new age identity on the YubiKey
age-plugin-yubikey --generate --slot 1
# Outputs recipient: age1yubikey1q...

# Encrypt to the YubiKey recipient
age -r age1yubikey1q... -o secrets/prod.age secrets/prod.txt

# Decrypt — requires physical YubiKey touch
age -d -i age-plugin-yubikey: secrets/prod.age
```

YubiKey-backed age identities are particularly useful for repository administrators who hold the master encryption key. Day-to-day CI pipelines use the shared symmetric git-crypt key, while the YubiKey-secured age identity protects the key export itself.

## Team Offboarding: Revoking Access Without Rekeying

When a team member leaves, git-crypt has no native revocation — you cannot simply remove their GPG key and have them locked out of future commits, because they already have a copy of the decrypted history. The correct procedure:

1. Remove the departing user's key blob from the repository:

```bash
# Find their GPG key fingerprint
gpg --list-keys departing@example.com
# FINGERPRINT: ABCD1234...

# Remove their key blob
rm .git-crypt/keys/default/0/ABCD1234.gpg
git add -A
git commit -m "Remove departing user from git-crypt access"
```

2. Rotate the symmetric key and all secrets it protects:

```bash
# Export current encrypted files as plaintext
git-crypt unlock
cp secrets/api-keys.json /tmp/api-keys-backup.json

# Re-initialize git-crypt with a new key
git-crypt init
git-crypt export-key ~/backup/git-crypt-key-v2.bin

# Re-add remaining team members
git-crypt add-gpg-user alice@example.com
git-crypt add-gpg-user bob@example.com

# Re-encrypt the files (they are already tracked by .gitattributes)
git add secrets/
git commit -m "Rekey git-crypt after team member departure"
```

3. Distribute the new key to CI/CD systems and remaining team members.
4. Rotate the actual secret values (API keys, passwords) as an independent step — assume the departing person has copies of the plaintext values regardless of repository access.

The combination of key rotation and secret rotation is the only complete offboarding. Repository access removal alone is insufficient.

## Related Reading

- [Privacy Tools Guide Hub](/privacy-tools-guide/)

Built by theluckystrike -- More at [zovo.one](https://zovo.one)
{% endraw %}
