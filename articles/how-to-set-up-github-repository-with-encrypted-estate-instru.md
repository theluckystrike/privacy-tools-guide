---
layout: default
title: "How to Set Up GitHub Repository with Encrypted Estate Instructions Using Age Encryption Tool"
description: "Learn how to securely store estate planning documents and sensitive instructions in a GitHub repository using age encryption. A practical guide for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-github-repository-with-encrypted-estate-instru/
categories: [guides, encryption, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Storing sensitive estate planning documents, recovery instructions, or digital legacy information requires careful consideration. While cloud storage services offer convenience, they often lack the granular control developers and privacy-conscious users need. Setting up a GitHub repository with age encryption provides version control, cross-device access, and military-grade encryption without relying on a single provider's infrastructure.

Age (ages file) is a modern encryption tool designed for simplicity and security. Unlike PGP, age uses modern cryptography (X25519 and ChaCha20-Poly1305) without the complexity of key servers or web-of-trust management. This makes it ideal for encrypting estate instructions that multiple trusted individuals might need to access.

## Prerequisites

Before setting up your encrypted repository, ensure you have:

- Git installed on your system
- Age encryption tool installed
- A GitHub account (free tier works fine)
- Terminal access with basic command-line knowledge

Install age on macOS with Homebrew:

```bash
brew install age
```

On Linux, use the package manager or download from GitHub:

```bash
sudo apt install age
```

Verify the installation:

```bash
age --version
```

## Generating Your Encryption Keys

Age supports two encryption methods: passphrase-based and key-pair encryption. For estate instructions, use key-pair encryption as it provides better security and allows for key rotation without re-encrypting everything.

Generate your key pair:

```bash
age-keygen -o ~/estate-keys/age.key
```

This creates two files: a private key (keep this secure, never commit it) and a public key. The public key looks like:

```
age1ql3z8tgnlqshwucehfj6hfcxxs5a9x7pn9fcvp6m3l6k2u3q0z8s7q
```

Store your private key securely. Consider using a password manager or hardware security key. Never commit the private key to any repository.

## Creating the GitHub Repository

Create a new private repository on GitHub:

1. Click "New repository" on GitHub
2. Name it `estate-encrypted` or similar
3. Select "Private" (important for sensitive documents)
4. Add a `.gitignore` for the key files

Clone it locally:

```bash
git clone git@github.com:yourusername/estate-encrypted.git
cd estate-encrypted
```

Create a `.gitignore` to prevent accidentally committing keys:

```bash
echo "*.key" >> .gitignore
echo "age.keys/" >> .gitignore
git add .gitignore
git commit -m "Add .gitignore for key files"
```

## Encrypting Estate Documents

Create your estate instructions file. This might include:

- Account recovery information
- Password vault master key location
- Instructions for next of kin
- Location of physical safe deposit keys
- Cryptocurrency wallet recovery phrases

Save this as `instructions.txt`. Now encrypt it using your public key:

```bash
age -r age1ql3z8tgnlqshwucehfj6hfcxxs5a9x7pn9fcvp6m3l6k2u3q0z8s7q \
    -o instructions.txt.age instructions.txt
```

This creates an encrypted file that only someone with the corresponding private key can decrypt. Remove the plaintext file:

```bash
rm instructions.txt
```

## Adding Multiple Recipients

For estate instructions, you might want multiple people to have access. Age supports multiple recipients:

```bash
age -r age1recipient-public-key -r age2recipient-public-key \
    -o instructions.txt.age instructions.txt
```

Each recipient with their corresponding private key can decrypt the file independently.

## Automating Encryption in Git Hooks

Set up a pre-commit hook to ensure you never accidentally commit plaintext files:

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# Check for unencrypted sensitive files
for file in $(git diff --cached --name-only --diff-filter=ACM); do
    if [[ "$file" == *.txt ]] && [[ ! "$file" == *.age ]]; then
        echo "Error: Plaintext file detected: $file"
        echo "Encrypt with: age -r <public-key> -o $file.age $file"
        exit 1
    fi
done
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

## Decrypting When Needed

When you or your trusted individuals need to read the instructions:

```bash
age -d -i ~/estate-keys/age.key -o decrypted.txt instructions.txt.age
```

This prompts for the passphrase protecting your private key and outputs the decrypted content.

## Rotating Keys

If you need to rotate encryption keys (change access, key compromise, etc.), re-encrypt all files with new keys:

```bash
# Decrypt with old key
age -d -i old-key.age -o content.txt content.txt.age

# Encrypt with new key
age -r new-public-key -o content.txt.age content.txt
```

Maintain a key rotation schedule and document which keys were valid during which time periods.

## Backup Strategy

Your encryption keys are the weak point in this system. Implement a proper backup strategy:

- Store a copy of the private key in a secure physical location (safe deposit box)
- Consider splitting the key using age's built-in functionality for additional security
- Document the recovery process for your trusted individuals

## Best Practices Summary

Use these practices when maintaining your encrypted estate repository:

- Always use private repositories for sensitive information
- Never commit private keys or plaintext sensitive files
- Use long, unique passphrases for key protection
- Test decryption regularly to ensure keys remain valid
- Document the decryption process for non-technical trusted individuals
- Consider using signed commits to verify your authorship
- Review and update encrypted documents annually

## Conclusion

Setting up a GitHub repository with age encryption provides a secure, version-controlled method for storing estate instructions and sensitive documents. The combination of modern encryption, git's history tracking, and GitHub's accessibility makes this approach practical for developers and power users who want full control over their digital legacy information.

The initial setup takes approximately 30 minutes, but provides years of reliable, encrypted storage with full audit trails. Your sensitive information remains readable only by those you explicitly authorize, regardless of what happens to your other accounts or services.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
