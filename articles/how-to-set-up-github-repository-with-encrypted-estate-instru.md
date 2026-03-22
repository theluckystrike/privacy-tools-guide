---
layout: default
title: "How To Set Up GitHub Repository With Encrypted Estate"
description: "Learn how to securely store sensitive estate planning documents, passwords, and financial information in a GitHub repository using encryption. This"
date: 2026-03-19
last_modified_at: 2026-03-19
author: "Privacy Tools Guide"
permalink: /how-to-set-up-github-repository-with-encrypted-estate-instru/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---
---
layout: default
title: "How To Set Up GitHub Repository With Encrypted Estate"
description: "Learn how to securely store sensitive estate planning documents, passwords, and financial information in a GitHub repository using encryption. This"
date: 2026-03-19
last_modified_at: 2026-03-19
author: "Privacy Tools Guide"
permalink: /how-to-set-up-github-repository-with-encrypted-estate-instru/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

Storing sensitive estate planning documents, password vaults, and financial instructions requires AES-256 encryption before they ever touch a cloud repository. This guide shows you how to set up a GitHub repository with encrypted files using age encryption, ensuring that even if your repository is compromised, your most sensitive information remains unreadable. We'll cover three production-ready approaches: age (the modern, Go-based encryption tool favored by the Tor Project), git-crypt (for Git-aware transparent encryption), and SOPS (for YAML/JSON-based secret management used by Mozilla and Discord). Each method has distinct trade-offs in key management, CI/CD integration, and recovery workflows that matter for estate planning where losing access means your heirs cannot recover anything.

## Key Takeaways

- **Encryption at rest in**: your repository ensures that even complete repository access by an attacker yields only useless ciphertext.
- **It uses X25519 for**: key exchange and ChaCha20-Poly1305 for encryption, meeting modern cryptographic standards while remaining approachable for non-cryptographers.
- **You can track when**: you added a new bank account, see the history of changes to your funeral preferences, and revert mistakes without physically managing multiple versions of spreadsheets or PDFs.
- **This provides tamper-resistant storage**: and requires physical presence to use.
- **Use LUKS (Linux) or**: FileVault (macOS) encryption with a strong passphrase separate from your everyday passwords.
- **Consider a time-locked approach**: where keys become accessible after a specified period of inactivity, or use a safe deposit box that your executor can access with proper documentation after your death.

## Table of Contents

- [Why Encrypt Estate Documents in GitHub?](#why-encrypt-estate-documents-in-github)
- [Prerequisites](#prerequisites)
- [Troubleshooting](#troubleshooting)
- [Related Reading](#related-reading)

## Why Encrypt Estate Documents in GitHub?

Estate planning documents contain some of the most sensitive information a person can store digitally: master passwords to password managers, bank account credentials, locations of safety deposit keys, social security numbers, medical directives, and funeral preferences. Storing unencrypted versions of these documents in any cloud service creates a single point of failure that hackers, data breaches, or government subpoenas can exploit. Encryption before commit transforms your GitHub repository from a liability into a secure, version-controlled vault that your executor or family members can access using keys you've separately provided.

GitHub's own security features protect repositories against unauthorized access, but they do not protect against account takeovers, compromised developer machines, or accidental public exposure. The 2020 GitHub OAuth token breach, the 2022 GitLab database theft, and countless individual account compromises demonstrate that cloud-hosted Git repositories are not impenetrable. Encryption at rest in your repository ensures that even complete repository access by an attacker yields only useless ciphertext.

Beyond security, using Git for estate documents provides powerful version control that paper documents cannot match. You can track when you added a new bank account, see the history of changes to your funeral preferences, and revert mistakes without physically managing multiple versions of spreadsheets or PDFs. The Git history becomes an audit trail of your estate planning evolution, valuable for resolving disputes among heirs or proving that specific instructions were added at specific times.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Method 1: Age Encryption (Recommended)

Age is a modern encryption tool written in Go, endorsed by the Tor Project for bridging relaying, and designed for simplicity without sacrificing security. It uses X25519 for key exchange and ChaCha20-Poly1305 for encryption, meeting modern cryptographic standards while remaining approachable for non-cryptographers. Age's design philosophy prioritates clear security boundaries, making it easier to reason about what is protected and what is not.

### Installing Age

On macOS, install age via Homebrew:

```bash
brew install age
```

On Linux, use your package manager:

```bash
sudo apt install age  # Debian/Ubuntu
sudo pacman -S age    # Arch Linux
```

Verify the installation:

```bash
age --version
```

### Creating Encryption Keys

Generate a recipient public key that will be embedded in your encrypted files:

```bash
age-keygen
```

This outputs something like:

```
# created: 2026-03-19T10:30:00-07:00
# public key: age1EXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLE
AGE-SECRET-KEY-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

The public key (starting with `age1`) is safe to share and can be embedded in configuration files. The secret key must be stored securely—ideally on multiple encrypted USB drives stored in separate physical locations, plus one copy with your attorney or executor. For estate purposes, generate separate keys for each person who needs access, and store all keys using a password manager or hardware security key.

### Encrypting Files

Encrypt a single file:

```bash
age -p -o estate-documents.txt.age -r age1EXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLE estate-documents.txt
```

The `-p` flag prompts for a passphrase as an additional layer of protection. Without `-p`, encryption relies solely on the recipient key. For estate documents, using both the recipient key and a passphrase provides defense in depth—if someone obtains your recipient key but not the passphrase, they still cannot read the files.

Encrypt an entire directory:

```bash
tar czf - documents/ | age -r age1EXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLE -o documents.tar.gz.age
```

### Decrypting Files

Decrypt a file:

```bash
age -d -o estate-documents.txt estate-documents.txt.age
```

If you used a passphrase with `-p`, age prompts for it. The decrypted output writes to the specified file or stdout if no output file is specified.

### Automating Encryption in Git

Create a Git filter to encrypt specific files automatically on commit and decrypt them on checkout. Add this to your repository's `.gitattributes`:

```
*.txt filter=agecrypt diff=agecrypt
*.md filter=agecrypt diff=agecrypt
*.pdf filter=agecrypt diff=agecrypt
```

Create a Python wrapper script at `.git/hooks/filter-age` that Git calls automatically:

```python
#!/usr/bin/env python3
import sys
import subprocess

# Load recipient keys from .age-recipients (one per line)
with open('.age-recipients', 'r') as f:
    recipients = [line.strip() for line in f if line.strip()]

if len(sys.argv) < 2:
    sys.exit(1)

action = sys.argv[1]
filename = sys.argv[2]

if action == 'clean':
    # Encrypt file contents
    import subprocess
    result = subprocess.run(
        ['age', '-r'] + recipients,
        input=sys.stdin.read().encode(),
        capture_output=True
    )
    sys.stdout.buffer.write(result.stdout)
elif action == 'smudge':
    # Decrypt file contents
    result = subprocess.run(['age', '-d'], input=sys.stdin.read().encode(), capture_output=True)
    sys.stdout.buffer.write(result.stdout)
```

Configure Git to use this filter:

```bash
git config filter.agecrypt.clean "python3 .git/hooks/filter-age clean"
git config filter.agecrypt.smudge "python3 .git/hooks/filter-age smudge"
```

Now files matching your `.gitattributes` patterns encrypt automatically when you `git add` them and decrypt when you `git checkout`. The plaintext never leaves your local machine, but GitHub stores only encrypted versions.

### Step 2: Method 2: Git-Crypt for Transparent Encryption

Git-crypt provides Git-aware encryption that works smoothly with standard Git workflows. Unlike age, which requires manual encryption and decryption commands, git-crypt encrypts files transparently—committed files appear as ciphertext in Git, but checkout automatically decrypts them if you possess the correct GPG key. This approach suits teams or families comfortable with GPG key management.

### Installing Git-Crypt

On macOS:

```bash
brew install git-crypt
```

On Linux:

```bash
sudo apt install git-crypt  # Debian/Ubuntu
```

### Initializing Git-Crypt

Navigate to your repository and initialize git-crypt:

```bash
cd estate-repository
git-crypt init
```

Git-crypt generates a symmetric key and stores it encrypted in `.git-crypt/.git-crypt-key`. You need GPG to protect this key.

### Configuring File Patterns

Edit `.gitattributes` to specify which files to encrypt:

```
secret-info/** filter=git-crypt diff=git-crypt
*.key filter=git-crypt diff=git-crypt
passwords/** filter=git-crypt diff=git-crypt
```

Files not matching these patterns remain unencrypted and are stored in Git as plain text.

### Adding Collaborators

Export your GPG public key and import it into the repository:

```bash
git-crypt add-gpg-user recipient@example.com
```

This command encrypts the git-crypt key using the recipient's GPG public key and commits it to the repository. When that recipient clones the repository and has the corresponding GPG secret key, git-crypt automatically decrypts files matching the filter patterns.

For estate planning, add each trusted person (executor, attorney, family members) using their GPG keys. Each person receives the encrypted git-crypt key tailored to their GPG identity.

### Using Git-Crypt

After configuration, git-crypt works transparently:

```bash
# Edit encrypted files normally
vim passwords/master.txt
git add passwords/master.txt
git commit -m "Update master password"
git push origin main
```

On collaborators' machines with the correct GPG key, files decrypt automatically on checkout. On machines without the key, files appear as binary ciphertext.

To lock encrypted files (prevent decryption):

```bash
git-crypt lock
```

To unlock:

```bash
git-crypt unlock
```

### Step 3: Method 3: SOPS for Encrypted Secret Management

SOPS (Secrets OPerationS), developed by Mozilla and used by companies like Discord and embedded in the Cloudflare CDN infrastructure, encrypts YAML, JSON, INI, and Terraform files while preserving their structure. Unlike age or git-crypt which encrypt entire files, SOPS encrypts individual values within files, making it ideal for configuration files that mix sensitive and non-sensitive data. For estate planning, SOPS excels when your documents include both public information (like beneficiary names) and sensitive data (like account numbers) in the same file.

### Installing SOPS

On macOS:

```bash
brew install sops age
```

On Linux, download the appropriate binary from the GitHub releases page:

```bash
curl -LO https://github.com/mozilla/sops/releases/download/v3.7.3/sops_v3.7.3_linux_amd64.tar.gz
tar xzf sops_v3.7.3_linux_amd64.tar.gz
sudo mv sops /usr/local/bin/
```

### Creating Encrypted Files

Create a YAML file with mixed sensitive and public data:

```yaml
# estate-config.yaml
executor:
  name: "Jane Executor"
  email: "jane@example.com"

accounts:
  - name: "Chase Checking"
    account_number: "ENC[age1EXAMPLEEXAMPLE...,chaseAccountNumber]"
    routing_number: "ENC[age1EXAMPLEEXAMPLE...,chaseRoutingNumber]"
    notes: "Joint account with spouse"

  - name: "Vanguard Retirement"
    account_number: "ENC[age1EXAMPLEEXAMPLE...,vanguardAccountNumber]"
    notes: "Roth IRA"
```

Notice how `account_number` and `routing_number` are wrapped in `ENC[...]` markers. These values will be encrypted while `name`, `email`, and `notes` (unless marked) remain readable.

### Encrypting with Age

Create a `.sops.yaml` configuration file:

```yaml
creation_rules:
  - age_recipients:
      - age1EXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLEEXAMPLE
    path_regex: \.yaml$
```

Now edit the file using `sops`:

```bash
sops estate-config.yaml
```

SOPS opens your default editor with the decrypted file. Edit values as needed, then save and exit. SOPS re-encrypts only the changed values and writes the encrypted output back to the file. The result looks like:

```yaml
executor:
  name: "Jane Executor"
  email: "jane@example.com"

accounts:
  - name: "Chase Checking"
    account_number: ENC[AES256_GCM,data:XXXXX,iv:XXXXX,type:str]
    routing_number: ENC[AES256_GCM,data:XXXXX,iv:XXXXX,type:str]
    notes: "Joint account with spouse"
```

### Decrypting Files

Decrypt a SOPS file for viewing:

```bash
sops -d estate-config.yaml
```

Or extract a specific value:

```bash
sops -d --extract '["accounts"][0]["account_number"]' estate-config.yaml
```

This ability to decrypt individual values without exposing the entire file makes SOPS powerful for CI/CD pipelines where only specific secrets need injection.

### Step 4: Key Management for Estate Planning

Encryption is only as strong as key management. For estate documents, key management has unique requirements: keys must survive you, be accessible to your executor, and remain secure against unauthorized access during your lifetime. The following strategies address these competing needs.

### Primary Key Storage

Your primary encryption key (for age) or GPG key (for git-crypt) should be stored in three locations using the 3-2-1 backup principle: three copies, on two different types of media, with one stored offsite. For encryption keys specifically, these copies should be:

1. **Hardware security key** (YubiKey or SoloKey) storing the key in its secure element. This provides tamper-resistant storage and requires physical presence to use.

2. **Encrypted USB drive** stored in a safe deposit box or with your attorney. Use LUKS (Linux) or FileVault (macOS) encryption with a strong passphrase separate from your everyday passwords.

3. **Paper backup** using a QR code or armored paper (waterproof, fireproof paper designed for long-term storage). Store this in a physically separate location from the USB drive.

### Executor Access

Your executor needs access to decryption keys, but giving them keys during your lifetime creates security risks. Consider a time-locked approach where keys become accessible after a specified period of inactivity, or use a safe deposit box that your executor can access with proper documentation after your death.

For SOPS or git-crypt with multiple recipients, you can add your executor's GPG key to the repository, but store their key encrypted with a passphrase that you provide separately (in your will, in a sealed envelope with your attorney, or in a separate secure location).

### Key Recovery Scenarios

Test your recovery procedures before relying on them. Create a test repository with sample encrypted files, then attempt recovery using only the backup keys. Document the exact steps your executor should follow, including:

- Which software to install and from which source
- Where to find the keys
- The exact commands to decrypt each type of file
- How to verify decryption succeeded (e.g., checking file hashes or comparing known content)

Store this documentation alongside the keys or with your estate planning documents.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to set up github repository with encrypted estate?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}

## Related Reading

- [Github Pull Request Workflow For Distributed Teams](/privacy-tools-guide/github-pull-request-workflow-for-distributed-teams/)
- [How To Create Anonymous Github Account For Open Source Contr](/privacy-tools-guide/how-to-create-anonymous-github-account-for-open-source-contr/)
- [Business Email Privacy: How to Set Up Encrypted Email.](/privacy-tools-guide/business-email-privacy-how-to-set-up-encrypted-email-for-com/)
- [How to set up encrypted emergency access your family can](/privacy-tools-guide/encrypted-emergency-access-setup-family-password-recovery/)
- [Set Up Dead Man's Switch Using Cron Job to Release Encrypted](/privacy-tools-guide/how-to-set-up-dead-mans-switch-using-cron-job-to-release-enc/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
