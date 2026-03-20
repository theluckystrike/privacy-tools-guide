---
layout: default
title: "How to Encrypt Git Repos with git-crypt and age"
description: "Full guide to encrypting sensitive files in Git repos using git-crypt and age. Team key management, CI/CD integration, and GPG-free workflows covered."
date: 2026-03-20
author: theluckystrike
permalink: /how-to-encrypt-git-repos-with-git-crypt-and-age/
categories: [guides]
reviewed: true
score: 8
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

## Related Reading

- [Privacy Tools Guide Hub](/privacy-tools-guide/)

Built by theluckystrike -- More at [zovo.one](https://zovo.one)
{% endraw %}
