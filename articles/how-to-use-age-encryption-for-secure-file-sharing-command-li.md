---
layout: default
title: "How to Use Age Encryption for Secure File Sharing: Command Line Guide"
description: "A practical command line guide for using age encryption to securely share files. Learn key generation, encryption, decryption, and advanced workflows for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-age-encryption-for-secure-file-sharing-command-li/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Age provides a straightforward way to encrypt files for secure sharing without the complexity of traditional tools. This guide covers practical command-line workflows for sharing encrypted files safely with others.

## Why Age for File Sharing

Age (pronounced like the English word "age") is a modern encryption tool designed for simplicity. Unlike PGP, age requires no key servers, no web of trust, and no configuration files. The tool uses X25519 elliptic curve cryptography for key exchange and ChaCha20-Poly1305 for symmetric encryption—modern, well-audited algorithms with a small, understandable codebase.

The primary advantage for file sharing is that recipients only need their own age key pair (or even just a passphrase) to decrypt files you send them. There's no need to coordinate key exchange through complex channels.

## Installing Age

Install age on macOS with Homebrew:

```bash
brew install age
```

On Linux, download the binary from the GitHub releases:

```bash
wget https://github.com/FiloSottile/age/releases/download/v1.2.1/age-v1.2.1-linux-amd64.tar.gz
tar -xzf age-v1.2.1-linux-amd64.tar.gz
sudo mv age/age-keygen /usr/local/bin/
```

On Windows, use the Windows Package Manager:

```bash
winget install FiloSottile.age
```

Verify the installation:

```bash
age-keygen
```

This generates a new identity file showing both the private key (which you keep secret) and the recipient public key (which you share with others).

## Generating and Managing Keys

Create a new age key pair:

```bash
age-keygen
```

The output looks like:

```
# created: 2026-03-16T10:30:00Z
# public key: age1qwerasdfzxcv1234567890qwertyuiop1234567890qwertyuiop12345678
AGE-SECRET-KEY-1QWERASDFZXCV1234567890QWERTYUIOP1234567890QWERTYUIOP12345678
```

Save the public key (starting with `age1`) for sharing with anyone who needs to encrypt files for you. Keep the secret key absolutely private—anyone with this key can decrypt your files.

For convenience, store your identity in a file:

```bash
age-keygen -o ~/.age-identity
```

Set restrictive permissions on your identity file:

```bash
chmod 600 ~/.age-identity
```

## Encrypting Files for Specific Recipients

To encrypt a file for someone else (or yourself), use their public key:

```bash
age -r age1qwerasdfzxcv1234567890qwertyuiop1234567890qwertyuiop12345678 \
    -o sensitive.zip.age sensitive.zip
```

This creates `sensitive.zip.age`—an encrypted file only the holder of the corresponding private key can decrypt.

You can encrypt for multiple recipients simultaneously:

```bash
age -r age1recipient1... -r age1recipient2... \
    -o shared.zip.age sensitive.zip
```

This creates a file both recipients can decrypt independently.

## Using Passphrase-Based Encryption

For situations where public key exchange isn't practical, age supports passphrase encryption:

```bash
age -p -o confidential.txt.age confidential.txt
```

Age will prompt for a passphrase (with confirmation). The output file uses scrypt for key derivation from the passphrase.

To customize the scrypt work factor, adjust the memory cost:

```bash
age -p --scrypt-cost 17 -o secured.zip.age secured.zip
```

Higher values increase computational cost for brute-force attacks but slow legitimate decryption slightly.

## Decrypting Files

Decrypt a file using your identity file:

```bash
age -d -i ~/.age-identity -o original.pdf protected.pdf.age
```

Decrypt a passphrase-protected file:

```bash
age -d -o original.txt locked.txt.age
```

The tool prompts for the passphrase used during encryption.

## Practical File Sharing Workflows

### Sending Encrypted Files via Email

Encrypt first, then attach to email:

```bash
age -r recipient-public-key -o invoice-2026-03.age invoice-2026-03.pdf
```

Send only the encrypted file. The recipient downloads and decrypts locally.

### Using File Transfer Services Safely

Upload encrypted files to any cloud service:

```bash
age -r my-public-key -o backup-20260316.age \
    -p -o documents.age documents/
tar -czvf - documents/ | age -r my-public-key -o backup.tar.age
```

Even if the cloud service is compromised, your files remain encrypted.

### Secure Backup to Untrusted Storage

Create encrypted backups to any location:

```bash
tar czf - ~/important-docs | age -r age1mypublickey... \
    -o ~/backups/docs-$(date +%Y%m%d).tar.age
```

Store the encrypted archive anywhere without exposing plaintext.

### Batch Encryption for Multiple Files

Encrypt a directory tree:

```bash
tar cvf - /path/to/directory | age -r public-key -o archive.age
```

Or encrypt individual files with a consistent naming pattern:

```bash
for file in *.pdf; do
    age -r "$PUBLIC_KEY" -o "${file%.pdf}.age" "$file"
done
```

## Automating Age in Scripts

Integrate age into automation pipelines:

```bash
#!/bin/bash
# Decrypt configuration file for application startup

if [ -f "$APP_DIR/secrets.age" ]; then
    age -d -i "$APP_DIR/identity" -o "$APP_DIR/secrets.env" "$APP_DIR/secrets.age"
    source "$APP_DIR/secrets.env"
    rm "$APP_DIR/secrets.env"  # Clean up plaintext from disk
fi
```

For CI/CD environments, use identity files stored as secrets:

```bash
age -d -i <(echo "$AGE_SECRET_KEY") -o config.json config.json.age
```

## Security Considerations

Age encrypts file contents but not filenames. If filename sensitivity matters, rename files before encryption:

```bash
mv sensitive-document.pdf $(date +%s).age
age -r public-key -o $(date +%s).age.age $(date +%s).age
```

The encrypted output includes the original filename during decryption—be aware of this metadata.

Age does not support key revocation. If a key is compromised, you must re-encrypt any files with a new key. Plan key management accordingly.

For high-security scenarios, consider encrypting in multiple layers:

```bash
age -r primary-key -o layer1.age file.txt
age -r secondary-key -p -o final.age layer1.age
```

This requires both the primary key and passphrase to access the original file.

## Troubleshooting Common Issues

If decryption fails with "no matching identity," verify your identity file contains the correct private key:

```bash
cat ~/.age-identity
```

Ensure the file permissions are correct—age refuses to use identity files readable by other users.

For "invalid key format" errors, confirm you're using the correct public key format. Age public keys always start with `age1`.

When passphrase decryption fails, there is no recovery mechanism. Lost passphrases cannot be brute-forced within any reasonable timeframe.

## Conclusion

Age provides an elegant solution for secure file sharing through its minimalist design and strong cryptography. The command-line interface integrates naturally into existing workflows, whether you're encrypting a single document or automating bulk operations. Combined with secure key management practices, age enables safe file sharing without the overhead of traditional encryption tools.

Start by generating a key pair with `age-keygen`, then experiment with encrypting and decrypting files. Build from simple single-file operations toward automated pipelines as your requirements grow.


## Related Reading

- [Age Encryption Tool Tutorial for Developers: Complete Guide](/privacy-tools-guide/age-encryption-tool-tutorial-developers/)
- [How to Encrypt Files Before Cloud Upload: Privacy Guide](/privacy-tools-guide/how-to-encrypt-files-before-cloud-upload/)
- [Best Encrypted Backup Solution for Developers: 2026](/privacy-tools-guide/best-encrypted-backup-solution-for-developers/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
