---
layout: default
title: "How to Use Age Encryption for File Sharing 2026"
description: "age (Actually Good Encryption) tool guide. CLI examples, integration with other tools, comparison to GPG, key management."
date: 2026-03-21
author: Privacy Tools Guide
permalink: /how-to-use-age-encryption-for-file-sharing-2026/
categories: [guides]
tags: [privacy-tools-guide, encryption]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
slug: how-to-use-age-encryption-for-file-sharing-2026
published: true
---

--END AGE ENCRYPTED FILE-----
```

Paste this into email or chat. Recipient decrypts:
```bash
echo "-----BEGIN AGE ENCRYPTED FILE-----
YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUx..." | age -d -i ~/.age/key.txt
```

Step 7 - Age Encryption with Passphrase (No Keys)

age can encrypt with just a passphrase (no public keys needed). Useful for personal backups.

```bash
Encrypt
age -p backup.tar.gz > backup.tar.gz.age
Prompts - Enter passphrase

Decrypt
age -d -i - backup.tar.gz.age > backup.tar.gz
Prompts - Enter passphrase
```

The `-p` flag means "use passphrase instead of keys." This is simpler than GPG for backups (no key management), but less secure if the passphrase is weak.

Step 8 - Integration with Other Tools

SSH Key Import

If you have a SSH key pair, you can use it with age directly (no need to generate separate age keys):

```bash
age -r ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... file.txt > file.txt.age
```

Encrypt with someone's SSH public key (they can decrypt with their SSH private key).

```bash
Decrypt using SSH key
age -d -i ~/.ssh/id_ed25519 file.txt.age
```

This is powerful - share SSH public keys instead of age public keys.

Encrypt within scripts

Bash script to encrypt a directory:

```bash
#!/bin/bash

PUBKEY="age1qs8g9p0qx5q2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3"
PRIVKEY="$HOME/.age/key.txt"

Encrypt all .log files
for file in *.log; do
 age -r "$PUBKEY" "$file" > "$file.age"
 echo "Encrypted $file"
done

Decrypt all .log.age files
for file in *.log.age; do
 age -d -i "$PRIVKEY" "$file" > "${file%.age}"
 echo "Decrypted $file"
done
```

Docker integration

Encrypt secrets before building Docker images:

```dockerfile
FROM alpine:latest

Copy encrypted config
COPY app-config.yml.age /app/

Decrypt at runtime (key passed as build arg)
RUN apk add age
RUN echo "AGE-SECRET-KEY-1..." | age -d -i - /app/app-config.yml.age > /app/app-config.yml

CMD ["app"]
```

Kubernetes secrets

Encrypt YAML before committing:

```bash
Encrypt secret
age -r age1my_pubkey secret.yaml > secret.yaml.age
git add secret.yaml.age

Apply secret (decrypt in CI pipeline before kubectl apply)
age -d -i ~/.age/key.txt secret.yaml.age | kubectl apply -f -
```

Step 9 - Rotate Keys and Revocation

Rotate to a new key

1. Generate a new key pair:
 ```bash
 age-keygen > ~/.age/new-key.txt
 ```

2. Re-encrypt old files with new public key:
 ```bash
 age -d -i ~/.age/old-key.txt secrets.age | age -r age1_new_pubkey > secrets.age
 ```

3. Share new public key with colleagues; discard old key.

Revoke a key

If a key is compromised:
1. Stop using it immediately.
2. Generate a new key.
3. Re-encrypt all files with the new key.
4. Delete the old key file.

age doesn't have a revocation system (like GPG), so manual rotation is necessary.

Comparison - Age vs GPG for Different Use Cases

Use age for:
- File sharing between colleagues.
- Encrypting backups before cloud upload.
- Protecting secrets in version control.
- Automation scripts (faster, simpler syntax).
- New projects where GPG compatibility isn't required.

Use GPG for:
- Signing Git commits (well-integrated).
- Email encryption (widely supported).
- Projects that require GPG compatibility.
- Organizations with existing GPG infrastructure.

For most modern workflows, age is the better choice.

Troubleshooting

Error - "could not decrypt: wrong passphrase"
- You're using the wrong secret key. Check `-i ~/.age/key.txt` path.
- Or the file wasn't encrypted for your public key.

Error - "unknown recipient"
- The public key in `-r` is invalid. Check the format (should start with `age1`).

Error - "secret key is invalid"
- The key file is corrupted. Check `~/.age/key.txt` exists and is readable.

File won't decrypt
- The key used to encrypt might be different from the key you're using to decrypt.
- You encrypted with Bob's key, but you're trying to decrypt with your own.

Security Considerations

1. Never commit secret keys to Git. Add `~/.age/key.txt` to `.gitignore` globally:
 ```bash
 echo "~/.age/key.txt" >> ~/.gitignore_global
 git config --global core.excludesfile ~/.gitignore_global
 ```

2. Protect key files. Set proper permissions:
 ```bash
 chmod 600 ~/.age/key.txt
 ```

3. Use strong passphrases (if encrypting keys with a passphrase). 20+ characters, mixed case, numbers, symbols.

4. Back up keys. Store a backup of your secret key in a secure location (encrypted USB, secure vault). If you lose it, you can't decrypt files.

5. Verify public keys. For critical sharing, verify the public key with the recipient out-of-band (call them, check their website, compare in person).

Frequently Asked Questions

How long does it take to use age encryption for file sharing?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [How To Use Age Encryption For Secure File Sharing Command](/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
- [Age Encryption Tool Tutorial for Developers](/age-encryption-tool-tutorial-developers/)
- [Best Accessible Encrypted File Sharing Tool for Users With Cognitive Impairments 2026](/best-accessible-encrypted-file-sharing-tool-for-users-with-c/)
- [How to Set Up age Encryption for Teams](/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [Best AI Assistant for Creating Playwright Tests for File](https://bestremotetools.com/best-ai-assistant-for-creating-playwright-tests-for-file-upl/)

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
