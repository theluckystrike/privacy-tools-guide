---
layout: default
title: "Secure Code Signing Setup for Developers"
description: "Set up GPG commit signing, sigstore for containers, and reproducible build verification to prove code authenticity in your supply chain"
date: 2026-03-22
author: theluckystrike
permalink: /secure-code-signing-setup-for-developers/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Code signing proves that a commit, release binary, or container image came from a specific key holder and has not been tampered with. This matters for supply chain security — attackers increasingly target the build pipeline rather than the running software.

## Part 1: GPG Commit Signing

### Generate a Signing Key

```bash
# Generate Ed25519 key
gpg --expert --full-generate-key
# Choose: (9) ECC and ECC → (1) Curve 25519
# Expiry: 2y

gpg --list-secret-keys --keyid-format LONG
# sec   ed25519/3AA5C34371567BD2 2026-03-22 [SC]
```

### Configure Git

```bash
git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Verify a signed commit
git log --show-signature -1
```

### Publish Your Key

```bash
gpg --keyserver keys.openpgp.org --send-keys 3AA5C34371567BD2

# Or add to GitHub Settings > SSH and GPG keys
gpg --armor --export 3AA5C34371567BD2
```

### Verify Commits

```bash
git verify-commit HEAD
git verify-tag v1.2.0

# Log with signature status
git log --pretty="format:%H %G? %GS %aN"
# G = good, B = bad, U = unknown, N = no signature
```

## Part 2: Signing Release Binaries

```bash
# Sign a binary
gpg --armor --detach-sign --local-user 3AA5C34371567BD2 \
  myapp-v1.0.0-linux-amd64

# Create and sign checksum file
sha256sum myapp-v1.0.0-linux-amd64 > myapp-v1.0.0-SHA256SUMS
gpg --armor --detach-sign --local-user 3AA5C34371567BD2 \
  myapp-v1.0.0-SHA256SUMS

# Publish: binary + .asc + SHA256SUMS + SHA256SUMS.asc
```

Users verify:
```bash
gpg --keyserver keys.openpgp.org --recv-keys 3AA5C34371567BD2
gpg --verify myapp-v1.0.0-SHA256SUMS.asc myapp-v1.0.0-SHA256SUMS
sha256sum --check myapp-v1.0.0-SHA256SUMS
```

## Part 3: Container Image Signing with Cosign

### Install Cosign

```bash
curl -sLO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign
```

### Generate Key Pair and Sign

```bash
cosign generate-key-pair
# Creates cosign.key and cosign.pub

docker push ghcr.io/yourorg/myapp:v1.0.0
cosign sign --key cosign.key ghcr.io/yourorg/myapp:v1.0.0
```

### Verify

```bash
cosign verify --key cosign.pub ghcr.io/yourorg/myapp:v1.0.0
```

### Keyless Signing with GitHub Actions

```yaml
# .github/workflows/sign.yml
name: Build and Sign

on:
  push:
    tags: ['v*']

permissions:
  id-token: write
  packages: write
  contents: read

jobs:
  build-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push image
        uses: docker/build-push-action@v5
        id: build
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      - name: Sign image (keyless)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

Keyless verification:
```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/yourorg/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/yourorg/myapp:v1.0.0
```

## Part 4: SSH Key Signing (Git 2.34+)

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

echo "your@email.com namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" \
  >> ~/.config/git/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers

git verify-commit HEAD
```

## Part 5: Protecting the Signing Key

```bash
# Backup GPG key (encrypted)
gpg --export-secret-keys --armor 3AA5C34371567BD2 | \
  gpg --symmetric --armor -o signing-key-backup.gpg.asc

# Store cosign.key in secrets manager
vault kv put secret/cosign-key key=@cosign.key

# Hardware key: use a YubiKey as a GPG smartcard
# gpg --card-edit → admin → generate
```

## Enforcing Signed Commits on GitHub

Signing commits is only half the solution. You also need to enforce that unsigned commits cannot be merged into protected branches.

**Branch protection rule (GitHub UI):**

```
Repository → Settings → Branches → Add branch protection rule
Branch name pattern: main
✅ Require signed commits
✅ Require status checks to pass before merging
```

**Verify policy in GitHub Actions:**

```yaml
# .github/workflows/verify-signatures.yml
name: Verify commit signatures

on:
  pull_request:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check all PR commits are signed
        run: |
          git log origin/main..HEAD --format="%H %G?" | while read hash status; do
            if [ "$status" != "G" ] && [ "$status" != "U" ]; then
              echo "UNSIGNED commit: $hash (status: $status)"
              exit 1
            fi
          done
          echo "All commits signed"
```

For teams, store trusted public keys in the repository and verify against them:

```bash
# .github/trusted-signers
# One entry per trusted contributor
alice@example.com namespaces="git" ssh-ed25519 AAAA...
bob@example.com namespaces="git" ssh-ed25519 AAAA...
```

## Key Rotation and Expiry Management

Signing keys should expire. Plan for rotation before expiry to avoid gaps:

```bash
# Check key expiry
gpg --list-keys --with-colons | grep "^pub" | cut -d: -f7
# Output is Unix timestamp of expiry; compare to $(date +%s)

# Extend expiry before it lapses
gpg --edit-key 3AA5C34371567BD2
gpg> expire
# Set new expiry: 2y
gpg> save

# Re-publish updated key
gpg --keyserver keys.openpgp.org --send-keys 3AA5C34371567BD2
```

For cosign keys, rotation means:
1. Generate a new key pair: `cosign generate-key-pair`
2. Sign new images with the new key
3. Verify old images still pass with the old public key (do not discard it)
4. Update CI secrets: replace `COSIGN_PRIVATE_KEY` with new key value
5. Document the rotation date and keep the old `cosign.pub` for historical verification

A key rotation schedule — annually at minimum, quarterly for high-security projects — should be part of your supply chain security policy, not a reaction to a compromise.

## Related Reading

- [How to Set Up Authentik for Identity Management](/privacy-tools-guide/how-to-set-up-authentik-for-identity-management/)
- [How to Audit Docker Images for Vulnerabilities](/privacy-tools-guide/how-to-audit-docker-images-for-vulnerabilities/)
- [Securing Docker Containers Best Practices](/privacy-tools-guide/securing-docker-containers-best-practices/)
- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
- [AI Code Generation Quality for Java Spring Security](https://theluckystrike.github.io/ai-tools-compared/ai-code-generation-quality-for-java-spring-security-configur/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
