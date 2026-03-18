---

layout: default
title: "How to Use AGE Encryption for Secure File Sharing: Command Line Guide"
description: "A practical guide to using age encryption for secure file sharing via command line. Learn installation, key generation, encryption, decryption, and automation patterns."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-age-encryption-for-secure-file-sharing-command-li/
categories: [guides]
---

{% raw %}

 AGE is a modern, minimalist encryption tool designed for simplicity and security. Created by Filippo Valsorda, age provides a clean command-line interface for encrypting files using modern cryptography. Unlike older tools that rely on outdated algorithms or complex key management, age uses ChaCha20-Poly1305 for symmetric encryption and supports SSH keys for recipient authentication. This guide walks you through installing age, generating keys, encrypting and decrypting files, and integrating it into your workflow for secure file sharing.

## Installing AGE

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

## Generating Encryption Keys

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

## Encrypting Files

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

## Decrypting Files

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

## Automation Patterns

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

## Conclusion

Age provides a clean, modern interface for file encryption that fits naturally into command-line workflows. Its minimal design means less configuration and fewer opportunities for misconfiguration. Install age, generate a key, and start encrypting sensitive files in minutes.

For developers sharing credentials, deploying to production environments, or backing up sensitive data, age offers a reliable solution without the complexity of traditional encryption tools.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
