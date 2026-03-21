---
title: How to Use Age Encryption for File Sharing 2026
slug: how-to-use-age-encryption-for-file-sharing-2026
description: age (Actually Good Encryption) tool guide. CLI examples, integration with other tools, comparison to GPG, key management.
author: Privacy Tools Guide
published: true
reviewed: true
score: 8
voice-checked: true
intent-checked: true
date: 2026-03-21
---

{% raw %}

age is a modern encryption tool that solves problems GPG has struggled with for decades. It's fast, simple, and designed for the way people actually share files today: over cloud storage, version control, and messaging apps. If you've ever fumbled with GPG commands (keyrings, trust models, armor format), age feels like a breath of fresh air.

This guide covers age command-line usage, integration with Git, automation, key management, and comparison to GPG. By the end, you'll encrypt files, share with others, and integrate age into your workflow.

## Age vs GPG

GPG is old and powerful but has UX problems. age is new and intentionally simple.

| Feature | GPG | age |
|---------|-----|-----|
| Ease of use | Hard | Easy |
| Installation | Complex | Single binary |
| Command length | 20-character commands | 3-4 character commands |
| Key format | Armor (ASCII, 50+ lines) | Bech32 (single line) |
| Learning curve | Steep | 30 minutes |
| Production use | Yes | Yes |
| Decryption speed | Slow | Fast |
| Supported by CLI tools | Yes | Growing |

**GPG:** Industry standard, widely compatible, heavily used in Git.

**age:** Modern simplicity, perfect for file sharing and backups, better UX.

## Installation

### macOS
```bash
brew install age
```

### Linux
```bash
sudo apt-get install age  # Debian/Ubuntu
sudo yum install age      # RedHat/CentOS
```

### Windows
```bash
choco install age         # Chocolatey
# or download from https://github.com/FiloSottile/age/releases
```

### Verify installation
```bash
age --version
# age 1.1.1
```

## Key Generation

Generate a key pair:
```bash
age-keygen > ~/.age/key.txt
```

Output:
```
# created: 2026-03-21T10:30:00Z
# public key: age1qs8g9p0qx5q2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3
AGE-SECRET-KEY-1QQS8G9P...AAAAAAAAAAAAAAAAAAAAAAAAA
```

The file contains:
- **Public key** (age1...): Share this; it's used to encrypt files for you.
- **Secret key** (AGE-SECRET-KEY-1...): Keep secret; it's used to decrypt files.

**Key management:**
```bash
# Protect the key file with a passphrase
chmod 600 ~/.age/key.txt

# Extract public key (useful for sharing)
age-keygen -y ~/.age/key.txt > ~/.age/key.pub.txt
```

The public key is shareable; distribute it freely (it's on your business card, GitHub profile, team wiki).

## Basic Encryption and Decryption

### Encrypt a file
```bash
age -r age1qs8g9p0qx5q2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3 document.pdf > document.pdf.age
```

**Syntax:**
- `-r` (recipient): Whose public key to use (yours, in this case).
- `document.pdf`: File to encrypt.
- `> document.pdf.age`: Save encrypted output (convention: `.age` extension).

File is encrypted in seconds; no bloat.

### Decrypt a file
```bash
age -d -i ~/.age/key.txt document.pdf.age > document.pdf
```

**Syntax:**
- `-d`: Decrypt (not encrypt).
- `-i ~/.age/key.txt`: Path to your secret key.
- `document.pdf.age`: Encrypted file.
- `> document.pdf`: Output decrypted content.

age prompts you for your key's passphrase (if one is set), then decrypts.

### Encrypt to multiple recipients
Share a file with a team:
```bash
age -r age1_alice_pubkey -r age1_bob_pubkey -r age1_carol_pubkey secret.txt > secret.txt.age
```

All three people can decrypt with their own secret keys. No key server needed.

## Practical Workflow: File Sharing

### Share with a colleague

**Step 1:** Get their public key (they send you `age1...`).

**Step 2:** Encrypt a file for them:
```bash
age -r age1_bob_pubkey confidential.xlsx > confidential.xlsx.age
```

**Step 3:** Share the file (send `.age` file via email, Slack, cloud storage—it's safe encrypted).

**Step 4:** They decrypt:
```bash
age -d -i ~/.age/key.txt confidential.xlsx.age > confidential.xlsx
```

Done. No GPG key servers, no fingerprint verification (though verification is an option).

### Share backup files to cloud storage

```bash
# Encrypt backup before uploading
tar czf backup.tar.gz /data
age -r age1my_pubkey backup.tar.gz > backup.tar.gz.age
aws s3 cp backup.tar.gz.age s3://my-bucket/

# Decrypt when needed
aws s3 cp s3://my-bucket/backup.tar.gz.age .
age -d -i ~/.age/key.txt backup.tar.gz.age | tar xzf -
```

The cloud storage provider never sees unencrypted data.

### Encrypt files in version control (Git)

Use git-age or manual workflow:

**Manual approach:**
```bash
# Encrypt sensitive config
age -r age1my_pubkey config.prod.yml > config.prod.yml.age

# Commit encrypted file, not plaintext
git add config.prod.yml.age
git rm config.prod.yml
git commit -m "Encrypt production config"

# To use, decrypt locally (don't commit plaintext)
age -d -i ~/.age/key.txt config.prod.yml.age > config.prod.yml
echo "config.prod.yml" >> .gitignore
```

**Better: Use a .gitattributes filter**
Create a `.gitattributes` file:
```
*.secret.yml filter=age diff=age
```

Configure Git:
```bash
git config filter.age.clean "age -r age1my_pubkey"
git config filter.age.smudge "age -d -i ~/.age/key.txt"
git config diff.age.textconv "age -d -i ~/.age/key.txt"
```

Now, when you commit `secrets.secret.yml`, Git automatically encrypts it. When you check it out, Git automatically decrypts it. The repo only contains encrypted files.

## Age with Armor (ASCII Format)

By default, age outputs binary. For pasting into text (emails, messages), use armor:

```bash
age -r age1my_pubkey --armor secret.txt > secret.txt.age
```

Output:
```
-----BEGIN AGE ENCRYPTED FILE-----
YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBvdUczSDg3YVN6RjJQRWgK
aVFrcjBFaVRzQWF3ZHdzUWpMdEFXTDQycm1SQVEvQWd1YU55Vm5Jd0UKLS1EJmV3
...
-----END AGE ENCRYPTED FILE-----
```

Paste this into email or chat. Recipient decrypts:
```bash
echo "-----BEGIN AGE ENCRYPTED FILE-----
YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUx..." | age -d -i ~/.age/key.txt
```

## Age Encryption with Passphrase (No Keys)

age can encrypt with just a passphrase (no public keys needed). Useful for personal backups.

```bash
# Encrypt
age -p backup.tar.gz > backup.tar.gz.age
# Prompts: Enter passphrase

# Decrypt
age -d -i - backup.tar.gz.age > backup.tar.gz
# Prompts: Enter passphrase
```

The `-p` flag means "use passphrase instead of keys." This is simpler than GPG for backups (no key management), but less secure if the passphrase is weak.

## Integration with Other Tools

### SSH Key Import

If you have an SSH key pair, you can use it with age directly (no need to generate separate age keys):

```bash
age -r ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... file.txt > file.txt.age
```

Encrypt with someone's SSH public key (they can decrypt with their SSH private key).

```bash
# Decrypt using SSH key
age -d -i ~/.ssh/id_ed25519 file.txt.age
```

This is powerful: share SSH public keys instead of age public keys.

### Encrypt within scripts

Bash script to encrypt a directory:

```bash
#!/bin/bash

PUBKEY="age1qs8g9p0qx5q2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3"
PRIVKEY="$HOME/.age/key.txt"

# Encrypt all .log files
for file in *.log; do
  age -r "$PUBKEY" "$file" > "$file.age"
  echo "Encrypted $file"
done

# Decrypt all .log.age files
for file in *.log.age; do
  age -d -i "$PRIVKEY" "$file" > "${file%.age}"
  echo "Decrypted $file"
done
```

### Docker integration

Encrypt secrets before building Docker images:

```dockerfile
FROM alpine:latest

# Copy encrypted config
COPY app-config.yml.age /app/

# Decrypt at runtime (key passed as build arg)
RUN apk add age
RUN echo "AGE-SECRET-KEY-1..." | age -d -i - /app/app-config.yml.age > /app/app-config.yml

CMD ["app"]
```

### Kubernetes secrets

Encrypt YAML before committing:

```bash
# Encrypt secret
age -r age1my_pubkey secret.yaml > secret.yaml.age
git add secret.yaml.age

# Apply secret (decrypt in CI pipeline before kubectl apply)
age -d -i ~/.age/key.txt secret.yaml.age | kubectl apply -f -
```

## Key Rotation and Revocation

### Rotate to a new key

1. Generate a new key pair:
   ```bash
   age-keygen > ~/.age/new-key.txt
   ```

2. Re-encrypt old files with new public key:
   ```bash
   age -d -i ~/.age/old-key.txt secrets.age | age -r age1_new_pubkey > secrets.age
   ```

3. Share new public key with colleagues; discard old key.

### Revoke a key

If a key is compromised:
1. Stop using it immediately.
2. Generate a new key.
3. Re-encrypt all files with the new key.
4. Delete the old key file.

age doesn't have a revocation system (like GPG), so manual rotation is necessary.

## Comparison: Age vs GPG for Different Use Cases

### Use age for:
- File sharing between colleagues.
- Encrypting backups before cloud upload.
- Protecting secrets in version control.
- Automation scripts (faster, simpler syntax).
- New projects where GPG compatibility isn't required.

### Use GPG for:
- Signing Git commits (well-integrated).
- Email encryption (widely supported).
- Projects that require GPG compatibility.
- Organizations with existing GPG infrastructure.

For most modern workflows, age is the better choice.

## Troubleshooting

**Error: "could not decrypt: wrong passphrase"**
- You're using the wrong secret key. Check `-i ~/.age/key.txt` path.
- Or the file wasn't encrypted for your public key.

**Error: "unknown recipient"**
- The public key in `-r` is invalid. Check the format (should start with `age1`).

**Error: "secret key is invalid"**
- The key file is corrupted. Check `~/.age/key.txt` exists and is readable.

**File won't decrypt**
- The key used to encrypt might be different from the key you're using to decrypt.
- Example: You encrypted with Bob's key, but you're trying to decrypt with your own.

## Security Considerations

1. **Never commit secret keys to Git.** Add `~/.age/key.txt` to `.gitignore` globally:
   ```bash
   echo "~/.age/key.txt" >> ~/.gitignore_global
   git config --global core.excludesfile ~/.gitignore_global
   ```

2. **Protect key files.** Set proper permissions:
   ```bash
   chmod 600 ~/.age/key.txt
   ```

3. **Use strong passphrases** (if encrypting keys with a passphrase). 20+ characters, mixed case, numbers, symbols.

4. **Back up keys.** Store a backup of your secret key in a secure location (encrypted USB, secure vault). If you lose it, you can't decrypt files.

5. **Verify public keys.** For critical sharing, verify the public key with the recipient out-of-band (call them, check their website, compare in person).

## Conclusion

age is simpler, faster, and more modern than GPG for file sharing and backup encryption. If you're starting a new project, encrypt files, or protect backups, age is the right choice. The learning curve is shallow (30 minutes), and the command-line syntax is intuitive. Integrate it into your workflows for SSH key encryption, Git secrets, cloud backups, and CI/CD pipelines.

For existing GPG setups, there's no urgent need to migrate. For new projects, age is the default.

{% endraw %}
