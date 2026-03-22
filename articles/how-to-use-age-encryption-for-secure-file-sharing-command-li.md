---
---
layout: default
title: "How To Use Age Encryption For Secure File Sharing Command"
description: "Install age (brew install age on macOS, apt install age on Linux), generate recipient keys with age-keygen, then encrypt files using age -r [public-key]"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-use-age-encryption-for-secure-file-sharing-command-li/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

Install age (`brew install age` on macOS, `apt install age` on Linux), generate recipient keys with `age-keygen`, then encrypt files using `age -r [public-key] file.txt > file.txt.age` and decrypt with `age -d file.txt.age`. For best results, share your public key through a trusted channel, have recipients encrypt their files using your key, and decrypt using your private key. Age is simpler than GPG, works with modern cryptography (ChaCha20-Poly1305), and requires no key servers or complex configuration.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Security Considerations](#security-considerations)
- [Comparison with GPG](#comparison-with-gpg)
- [Performance and Scalability](#performance-and-scalability)
- [Threat Model Analysis](#threat-model-analysis)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install AGE

The installation process varies by operating system, but most developers can install age with a single command. On macOS, use Homebrew:

```bash
brew install age
```

On Linux, age is available through most package managers. For Debian-based distributions:

```bash
sudo apt install age
```

For other platforms, download pre-built binaries from the [GitHub releases page](https://github.com/FiloSottile/age). Verify the installation by running:

```bash
age --version
```

You should see version information confirming a successful installation.

### Step 2: Generate Encryption Keys

Before encrypting files, you need to generate a keypair. AGE supports two types of keys: identity files (password-protected) and SSH keys. For most use cases, generating a dedicated identity file provides the best balance of security and convenience.

Generate a new identity file with:

```bash
age-keygen -o age-identity.txt
```

This command creates two outputs: a private key saved to `age-identity.txt` and a public key displayed in the terminal. The public key looks something like:

```
age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq
```

Protect your identity file. Anyone with access to this file can decrypt messages encrypted for your public key. Store it in a secure location, such as a password manager or encrypted directory.

If you prefer using existing SSH keys, age can generate the corresponding age public key from your SSH key:

```bash
age-keygen -y ~/.ssh/id_ed25519
```

This outputs the age-format public key derived from your SSH private key, allowing you to use existing credentials for age encryption.

### Step 3: Encrypt Files

With your keypair ready, encrypting files becomes straightforward. The basic syntax uses `age` with the `-p` flag for password-based encryption, or the `-r` flag for recipient-based encryption using public keys.

For password-based encryption:

```bash
age -p -o sensitive-data.tar.gz.age sensitive-data.tar.gz
```

This prompts for a passphrase and outputs the encrypted file. The `-p` flag adds password-based key derivation using Argon2, making the encryption resistant to brute-force attacks.

For recipient-based encryption using your public key:

```bash
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq -o document.pdf.age document.pdf
```

Replace the public key with your generated or derived key. You can specify multiple recipients by adding additional `-r` flags:

```bash
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq \
    -r age1anotherpublickeyhere -o file.tar.gz.age file.tar.gz
```

This allows multiple people to decrypt the same file using their respective private keys.

For encrypting directories, combine age with tar:

```bash
tar czf - /path/to/directory | age -p -o backup.tar.gz.age
```

### Step 4: Decrypt Files

Decryption requires the corresponding private key or the correct passphrase. The `age-decrypt` command handles both recipient-based and password-based encrypted files.

Using your identity file:

```bash
age-decrypt -i age-identity.txt document.pdf.age
```

This outputs the decrypted content to stdout. To save to a file, use the `-d` flag or redirect output:

```bash
age-decrypt -i age-identity.txt -o document.pdf document.pdf.age
```

For password-encrypted files:

```bash
age-decrypt -p -o original-file.txt encrypted-file.txt.age
```

The `-p` flag prompts for the passphrase used during encryption.

### Step 5: Automation Patterns

Integrating age into scripts and workflows requires understanding how to pass keys securely. Avoid hardcoding keys in scripts. Instead, use environment variables or file references with appropriate permissions.

Passing a key from an environment variable:

```bash
export AGE_PRIVATE_KEY=$(cat age-identity.txt)
age-decrypt -i /dev/stdin encrypted-file.age
```

For automated backups with age:

```bash
#!/bin/bash
BACKUP_FILE="backup-$(date +%Y%m%d).tar.gz"
tar czf "$BACKUP_FILE" /important/data
age -r age1publickey -o "$BACKUP_FILE.age" "$BACKUP_FILE"
rm "$BACKUP_FILE"
```

This script creates a compressed backup, encrypts it for a specific recipient, and removes the unencrypted original.

Combine age with scp or rsync for secure file transfer:

```bash
tar czf - /local/data | age -p | ssh user@server "cat > remote-data.tar.gz.age"
```

The receiving end then decrypts with the appropriate key or passphrase.

## Security Considerations

Age implements modern cryptographic primitives by default. The symmetric encryption uses ChaCha20-Poly1305, providing authenticated encryption that detects tampering. Key derivation uses Argon2id, resistant to both GPU and ASIC-based attacks when password-protected.

A few best practices improve your security posture. Never commit identity files to version control. Add them to `.gitignore` or use a separate secrets management approach. Rotate keys periodically, especially for shared or team encryption. When sharing files with others, verify public keys through a separate channel to prevent man-in-the-middle attacks.

For team usage, consider a key management strategy where each team member has their own key, and encrypted files list all team members as recipients. This maintains individual key control while enabling collaborative access.

## Comparison with GPG

Developers familiar with GPG might wonder why age exists. Age prioritizes simplicity and modern defaults over broad compatibility. GPG supports numerous algorithms, some outdated, and carries historical complexity from decades of development. Age chooses sane defaults—modern algorithms, no configuration required—and focuses on the most common use case: encrypting files for yourself or specific recipients.

For teams already using SSH, age's SSH key compatibility reduces the credential management burden. You can encrypt files using keys you already use for server authentication.

### Step 6: Age Cryptography Deep-Dive

Understanding age's cryptographic foundation ensures proper security assumptions:

### ChaCha20-Poly1305 Algorithm

Age uses ChaCha20-Poly1305, a modern AEAD cipher providing both confidentiality and authenticity:

```
ChaCha20: Stream cipher providing confidentiality
- 256-bit key
- 96-bit nonce (unique per encryption)
- Faster on CPUs without AES hardware acceleration
- No known practical cryptanalysis attacks

Poly1305: Polynomial authentication
- 128-bit authentication tag
- Detects any bit manipulation of ciphertext
- Timing-safe implementation (resistant to timing attacks)
```

This combination ensures ciphertext cannot be decrypted incorrectly without detection.

### Key Derivation Details

For password-based encryption, age uses Argon2id:

```
Parameters (default):
- Time cost: 3 iterations
- Memory cost: 64 MB
- Parallelism: 4 threads
- Output: 256-bit derived key

These defaults resist GPU attacks while remaining fast on modern hardware.
```

For public-key encryption, age uses Curve25519 for key agreement:

```
X25519: Elliptic curve Diffie-Hellman
- 256-bit security level
- Montgomery form (efficient point operations)
- No cofactor issues
- Widely considered cryptographically sound
```

### Step 7: Batch Encryption Operations

For processing many files:

```bash
#!/bin/bash
# Batch encrypt directory structure

RECIPIENT_KEY="age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq"
SOURCE_DIR="./sensitive-data"
OUTPUT_DIR="./encrypted-backup"

mkdir -p "$OUTPUT_DIR"

# Encrypt each file, preserving directory structure
find "$SOURCE_DIR" -type f | while read -r file; do
    # Calculate output path
    relative_path="${file#$SOURCE_DIR/}"
    output_path="$OUTPUT_DIR/$relative_path.age"

    # Create output directory
    mkdir -p "$(dirname "$output_path")"

    # Encrypt file
    age -r "$RECIPIENT_KEY" -o "$output_path" "$file"

    # Print progress
    echo "Encrypted: $relative_path"
done

# Create manifest of encrypted files
find "$OUTPUT_DIR" -type f -exec sha256sum {} \; > "$OUTPUT_DIR/manifest.sha256"
```

### Step 8: Integration with Backup Tools

Age integrates with backup workflows:

### Restic Backup with Age

```bash
# Setup restic with age encryption
restic init -r /mnt/backups -e age

# Create age key for backup
age-keygen -o restic-key.txt

# Set environment variable
export RESTIC_PASSWORD_COMMAND="age -d -i restic-key.txt < /tmp/restic.age"

# Backup with encryption
restic -r /mnt/backups backup /important/data

# Verify backup
restic -r /mnt/backups check

# Restore when needed
restic -r /mnt/backups restore latest --target /restore/location
```

### Tar + Age for Versioned Backups

```bash
#!/bin/bash
# Daily incremental backup with age

BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
RECIPIENT="age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq"
BACKUP_ROOT="/backups"

# Create tarball with modification time delta
find /data -type f -newer /tmp/last-backup-marker 2>/dev/null | \
    tar czf - -T - | \
    age -r "$RECIPIENT" -o "$BACKUP_ROOT/incremental-$BACKUP_DATE.tar.gz.age"

# Update marker for next run
touch /tmp/last-backup-marker

# List recent backups
ls -lh "$BACKUP_ROOT"/incremental-*.age | tail -5
```

### Step 9: Secure Key Sharing Protocols

Distributing keys securely is critical:

### Out-of-Band Verification

```bash
#!/bin/bash
# Share key through multiple channels

# Primary: encrypted email
echo "Your age public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq" | \
    mail -s "Your encryption key" recipient@example.com

# Secondary: SMS with fingerprint (short form)
FINGERPRINT=$(echo "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5eu9rq" | \
    sha256sum | cut -c1-16)
echo "Key fingerprint: $FINGERPRINT" | sms recipient

# Verify with voice call
# "For security, read back the fingerprint you received in SMS"
```

### Shamir's Secret Sharing for Keys

For high-security scenarios, split keys across trustees:

```bash
# Install ssss (Shamir's Secret Sharing)
brew install ssss

# Create key with 3-of-5 splitting
# Any 3 shares can recover the key
cat age-identity.txt | \
    ssss-split -t 3 -n 5 > key-shares.txt

# Distribute each share to separate trustee
split -n l/5 key-shares.txt share_

# To recover
cat share_00 share_01 share_02 | \
    ssss-combine > recovered-key.txt
```

## Performance and Scalability

Age handles large-scale encryption efficiently:

```bash
# Performance testing
# Age on 1GB file (modern CPU)
time age -r "$KEY" -o file.1gb.age file.1gb
# Typical: 0.5-1.5 seconds

# Parallel encryption of many files
find data -type f | \
    parallel age -r "$KEY" -o {}.age {}

# Memory usage: Minimal (~10MB regardless of file size)
# This is because age streams data rather than loading entirely
```

### Step 10: Decryption in Restricted Environments

Recovering files when tools are limited:

```bash
# On system without age installed:
# Compile minimal age decoder or use reference implementation

# Python implementation (partial, for testing)
import os
from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305

# This would require implementing full age format parsing
# For production, always compile/install proper age binary
```

### Step 11: Integration with Git for Encrypted Repositories

Store sensitive config in git with age encryption:

```bash
# Setup encrypted git attributes
echo "*.secret.txt diff=age" > .gitattributes

# Configure git filter
git config filter.age.clean "age -e -r $AGE_PUBLIC_KEY"
git config filter.age.smudge "age -d -i $AGE_IDENTITY_FILE"

# Track encrypted secrets
git add config.secret.txt
git commit -m "Add encrypted configuration"

# Developers with key can decrypt
git smudge .git/config.secret.txt > config.txt
```

## Threat Model Analysis

Age provides protection against specific threats:

```
Protected against:
- Network eavesdropping during file transfer
- Cloud storage provider accessing your files
- Attacker who gains access to encrypted files
- Key recovery without passphrase (password-based)

NOT protected against:
- Attacker with your private key
- Malware on your system during encryption/decryption
- Brute-force of weak passphrases
- Side-channel attacks on implementation
```

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to use age encryption for secure file sharing command?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Best Secure File Sharing Tools for Teams Handling.](/privacy-tools-guide/best-secure-file-sharing-tools-for-teams-handling-sensitive-data/)
- [How to Set Up Secure File Sharing for Sensitive Documents](/privacy-tools-guide/how-to-set-up-secure-file-sharing-for-sensitive-documents/)
- [Onionshare Secure File Sharing Over Tor Network Setup And Us](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Secure File Sharing Tools Comparison: E2E Encrypted.](/privacy-tools-guide/secure-file-sharing-tools-comparison/)
- [Age Encryption Tool Tutorial for Developers](/privacy-tools-guide/age-encryption-tool-tutorial-developers/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
```
```
{% endraw %}
