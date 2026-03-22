---
layout: default
title: "GPG Key Management Best Practices"
description: "How to structure GPG keys with a master key offline and subkeys in daily use, set expiry dates, back up keys securely, and handle key rotation and revocation"
date: 2026-03-21
author: theluckystrike
permalink: /gpg-key-management-best-practices/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, best-of]---
---
layout: default
title: "GPG Key Management Best Practices"
description: "How to structure GPG keys with a master key offline and subkeys in daily use, set expiry dates, back up keys securely, and handle key rotation and revocation"
date: 2026-03-21
author: theluckystrike
permalink: /gpg-key-management-best-practices/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, best-of]---

{% raw %}

Most people use GPG with a single keypair for everything: sign emails, encrypt files, authenticate to servers. When that key is compromised — and eventually, many are — they lose everything. The correct approach separates your master key (which certifies your identity) from the subkeys used for daily operations. The master key stays offline; the subkeys can be replaced without changing your public identity.

This guide covers the key management architecture used by security professionals.

## The Master Key / Subkey Architecture

**Master key (certification key)**:
- Has only the Certify capability
- Represents your persistent identity
- Stored on an air-gapped machine or hardware security key
- Used only to: certify subkeys, certify other people's keys, create revocation certificates

**Subkeys (daily use)**:
- Separate keys for: Signing, Encryption, Authentication
- Stored on your working machine or hardware key
- Can be individually revoked and replaced without changing your master key or public identity
- Set with expiry dates (1-2 years)

If your laptop is stolen: revoke the subkeys, generate new ones certified by your master key. Your correspondents continue to trust your master key fingerprint.

## Setting Up the Master Key

Do this on an air-gapped machine, or at minimum a machine that has never been connected to the internet:

```bash
# Use a temporary GPG home to keep this key separate during creation
export GNUPGHOME=$(mktemp -d)

# Create master key with ONLY the Certify capability
gpg --expert --full-gen-key
```

In the interactive dialog:
```
Please select what kind of key you want:
(8) RSA (set your own capabilities)
Current allowed actions: Sign Certify Encrypt

Toggle 's' to remove Sign capability
Toggle 'e' to remove Encrypt capability
Leave 'c' (Certify) only
```

Select:
- Key size: 4096
- Key does not expire (the master key should be long-lived; subkeys carry expiry)
- Your real name and email address

After creation, note the key ID:
```bash
gpg --list-keys
# Example: pub rsa4096 2026-03-21 [C] KEYID
```

## Adding Subkeys

Edit the master key to add subkeys:

```bash
gpg --expert --edit-key KEYID
```

Add three subkeys:

```
gpg> addkey
(8) RSA (set your own capabilities)
# Toggle to leave only Sign capability
# Key size: 4096
# Expiry: 1y (or 2y)

gpg> addkey
(8) RSA (set your own capabilities)
# Toggle to leave only Encrypt capability
# Key size: 4096
# Expiry: 1y

gpg> addkey
(8) RSA (set your own capabilities)
# Toggle to leave only Authenticate capability
# Key size: 4096
# Expiry: 1y

gpg> save
```

Verify the final structure:

```bash
gpg --list-keys --with-subkey-fingerprints KEYID
```

You should see:
```
pub   rsa4096 2026-03-21 [C]
      MASTERKEYFINGERPRINT
uid   [ultimate] Your Name <your@email.com>
sub   rsa4096 2026-03-21 [S] [expires: 2027-03-21]
      SIGNSUBKEYFINGERPRINT
sub   rsa4096 2026-03-21 [E] [expires: 2027-03-21]
      ENCRYPTSUBKEYFINGERPRINT
sub   rsa4096 2026-03-21 [A] [expires: 2027-03-21]
      AUTHSUBKEYFINGERPRINT
```

## Create a Revocation Certificate

Before doing anything else, create a revocation certificate for the master key:

```bash
gpg --gen-revoke KEYID > revocation-cert.asc
```

Store this file:
- Printed on paper in a physically secure location
- Encrypted on a hardware token or USB drive kept offline
- Multiple copies in separate physical locations

If your master key is ever compromised, importing this certificate to the keyserver is the only way to inform others that your key should no longer be trusted.

## Back Up the Master Key

Export the master key and subkeys:

```bash
# Export the full key (master + subkeys) — keep this completely offline
gpg --armor --export-secret-keys KEYID > master-and-subkeys.asc

# Export only the master key
gpg --armor --export-secret-keys --export-options export-minimal KEYID > master-only.asc

# Export public key (safe to publish)
gpg --armor --export KEYID > public-key.asc
```

Store `master-and-subkeys.asc` and `master-only.asc` on encrypted storage that stays offline. Write the passphrase down and store it separately from the key files.

## Export Subkeys for Daily Use

On your daily machine, you only need the subkeys — not the master key:

```bash
# Export only the subkeys (master key stub remains, but private master key is not present)
gpg --armor --export-secret-subkeys KEYID > subkeys-only.asc
```

Transfer `subkeys-only.asc` to your working machine and import it there:

```bash
# On your working machine
gpg --import subkeys-only.asc
gpg --import public-key.asc
```

Verify the setup — the `#` symbol indicates the master key is not present (only a stub):

```bash
gpg --list-secret-keys
# sec#  rsa4096  [C]   ← The # means master key is not present
# ssb   rsa4096  [S] [expires: ...]
# ssb   rsa4096  [E] [expires: ...]
# ssb   rsa4096  [A] [expires: ...]
```

This is the correct daily-use configuration. Signing, encryption, and authentication work normally through the subkeys. The master key (for certification operations) is unavailable unless you connect the offline backup.

## Setting Trust Level

After importing your key on the working machine:

```bash
gpg --edit-key KEYID
gpg> trust
# Select 5 = I trust ultimately (since it's your own key)
gpg> quit
```

## Extending or Renewing Subkey Expiry

When a subkey expires, it needs to be extended (preferred — keeps the same subkey fingerprint) or replaced (generates a new subkey with a new fingerprint).

Connect your offline master key backup, then:

```bash
# On the air-gapped machine (or after importing master key temporarily)
gpg --edit-key KEYID

gpg> key 1        # Select first subkey
gpg> expire       # Set new expiry
1y                # 1 more year
gpg> key 1        # Deselect
gpg> key 2        # Select second subkey
gpg> expire
1y
# Repeat for all subkeys
gpg> save
```

Re-export and re-publish the updated public key.

## Rotating a Compromised Subkey

If a subkey is compromised (laptop stolen, key file exposed):

```bash
# On the air-gapped machine with master key available
gpg --edit-key KEYID
gpg> key 1              # Select compromised subkey
gpg> revkey             # Revoke it
# Confirm and enter reason
gpg> addkey             # Add a replacement subkey with new key material
gpg> save

# Export new subkeys for daily use
gpg --armor --export-secret-subkeys KEYID > new-subkeys.asc

# Publish the revocation and new public key
gpg --send-keys KEYID
```

## Publishing Your Public Key

```bash
# Submit to multiple keyservers (they synchronize)
gpg --keyserver keys.openpgp.org --send-keys KEYID
gpg --keyserver keyserver.ubuntu.com --send-keys KEYID

# The fingerprint to share with others
gpg --fingerprint KEYID
```

## Related Reading

- [How to Manage PGP Keys Using a Hardware Security Key](/how-to-manage-pgp-keys-securely-using-hardware-security-key-/)
- [GPG Email Encryption Step by Step](/gpg-email-encryption-step-by-step/)
- [YubiKey Setup for Multiple Services Guide](/yubikey-setup-multiple-services-guide/)
- [AI Tools for Automating Cloud Security Compliance Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-automating-cloud-security-compliance-scanning-i/)
- [Best AI Tools for Container Security Scanning in Deployment](https://theluckystrike.github.io/ai-tools-compared/best-ai-tools-for-container-security-scanning-in-deployment-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Are free AI tools good enough for practices?**

Free tiers work for basic tasks and evaluation, but paid plans typically offer higher rate limits, better models, and features needed for professional work. Start with free options to find what works for your workflow, then upgrade when you hit limitations.

**How do I evaluate which tool fits my workflow?**

Run a practical test: take a real task from your daily work and try it with 2-3 tools. Compare output quality, speed, and how naturally each tool fits your process. A week-long trial with actual work gives better signal than feature comparison charts.

**Do these tools work offline?**

Most AI-powered tools require an internet connection since they run models on remote servers. A few offer local model options with reduced capability. If offline access matters to you, check each tool's documentation for local or self-hosted options.

**How quickly do AI tool recommendations go out of date?**

AI tools evolve rapidly, with major updates every few months. Feature comparisons from 6 months ago may already be outdated. Check the publication date on any review and verify current features directly on each tool's website before purchasing.

**Should I switch tools if something better comes out?**

Switching costs are real: learning curves, workflow disruption, and data migration all take time. Only switch if the new tool solves a specific pain point you experience regularly. Marginal improvements rarely justify the transition overhead.

{% endraw %}
