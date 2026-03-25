---
layout: default
title: "Email Encryption with GPG"
description: "Encrypt and decrypt emails with GPG from scratch. Covers key generation, key exchange, Thunderbird with OpenPGP, and key server publishing."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: gpg-email-encryption-step-by-step
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

GPG-encrypted email means only the intended recipient can read the message. not your email provider, not the recipient's email provider, not anyone who intercepts the message in transit. The tradeoff is that both sender and recipient need to have set up GPG and exchanged public keys.

This guide covers the complete workflow: generating a key pair, exchanging keys, encrypting and decrypting messages, and setting up Thunderbird for everyday use.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1 - Install GPG

```bash
Debian/Ubuntu
sudo apt install gnupg

Fedora/RHEL
sudo dnf install gnupg2

macOS (GPG Suite is easiest)
brew install gnupg
Or download GPG Suite from https://gpgtools.org

Windows - Gpg4win from https://www.gpg4win.org
```

Verify:

```bash
gpg --version
```

Step 2 - Generate Your Key Pair

```bash
gpg --full-generate-key
```

At the prompts:
- Key type: `(1) RSA and RSA` or `(9) ECC (sign and encrypt)`. choose ECC for modern security
- If ECC: select `Curve 25519`
- If RSA: use 4096 bits
- Expiration - `2y` (2 years). shorter is better; you can extend before expiry
- Name: your real name or pseudonym
- Email: the address you'll use for encrypted email
- Comment: leave blank or enter "GPG key"
- Passphrase: a strong passphrase, stored in your password manager

After generation, view your key:

```bash
gpg --list-secret-keys --keyid-format LONG
```

Output:
```
sec   ed25519/1A2B3C4D5E6F7890 2026-03-21 [SC] [expires: 2028-03-21]
      A1B2C3D4E5F6789012345678901234567890ABCD
uid                 [ultimate] Your Name <you@example.com>
ssb   cv25519/FEDCBA0987654321 2026-03-21 [E]
```

The long hex string starting with `1A2B...` is your key ID. The 40-character string is your fingerprint.

Step 3 - Export Your Public Key

Share this with anyone who wants to send you encrypted email:

```bash
Export in ASCII armor format
gpg --armor --export you@example.com > yourname-public-key.asc

View it
cat yourname-public-key.asc
```

The public key starts with `-----BEGIN PGP PUBLIC KEY BLOCK-----`. This is safe to share publicly. post it on your website, email it to contacts, publish it to a key server.

Step 4 - Back Up Your Private Key

Your private key is irreplaceable. Back it up to encrypted offline storage:

```bash
Export private key (keep this SECRET and ENCRYPTED)
gpg --armor --export-secret-key you@example.com > yourname-private-key.asc

Also export the revocation certificate (generated at key creation)
gpg --gen-revoke you@example.com > yourname-revoke.asc
```

Store both files on an encrypted USB drive or in an encrypted archive. If you lose the private key, you can't decrypt old messages. If someone else gets it, they can read everything encrypted to you.

Step 5 - Publish Your Key to a Key Server

Publishing makes it easy for others to find your key by email address:

```bash
Send to keys.openpgp.org (verifies email ownership before publishing)
gpg --keyserver keys.openpgp.org --send-keys YOUR_KEY_ID

Alternative - Ubuntu key server
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY_ID
```

After publishing to `keys.openpgp.org`, you'll receive a verification email. Confirm it so your key becomes searchable by email address.

You can also paste the public key directly on:
- Your personal website
- Your GitHub profile (`https://github.com/yourusername.gpg`)
- Email signatures

Step 6 - Import Someone Else's Public Key

To send an encrypted message, you need the recipient's public key.

From a key server:

```bash
Search by email
gpg --keyserver keys.openpgp.org --search-keys recipient@example.com

Import by key ID
gpg --keyserver keys.openpgp.org --recv-keys RECIPIENT_KEY_ID
```

From a file they sent you:

```bash
gpg --import recipient-public-key.asc
```

From GitHub:

```bash
curl https://github.com/theirusername.gpg | gpg --import
```

Verify the fingerprint matches what the recipient told you (over a separate channel):

```bash
gpg --fingerprint recipient@example.com
```

Step 7 - Encrypt a Message

```bash
Encrypt a file for a recipient (also sign it with your key)
gpg --recipient recipient@example.com \
    --sign \
    --armor \
    --encrypt message.txt

Output - message.txt.asc (the encrypted message)
```

The `--sign` flag attaches your digital signature. This proves the message came from you and hasn't been tampered with.

To encrypt for multiple recipients:

```bash
gpg --recipient alice@example.com \
    --recipient bob@example.com \
    --sign \
    --armor \
    --encrypt message.txt
```

The message is encrypted separately for each recipient's key. only Alice or Bob can decrypt it.

Step 8 - Decrypt a Message

```bash
Decrypt (will prompt for your passphrase)
gpg --decrypt message.txt.asc

Or redirect to a file
gpg --decrypt --output message.txt message.txt.asc
```

GPG automatically detects which private key to use based on the recipient field in the encrypted message.

Step 9 - Set Up Thunderbird with OpenPGP

Thunderbird 78+ has native OpenPGP support. no Enigmail required.

1. Import your key into Thunderbird:
 - Account Settings > End-To-End Encryption > Add Key
 - Choose "Use your external key" or import the `.asc` file

2. Set default behavior:
 - Require encryption: enable if all your contacts use GPG
 - Sign unencrypted messages: enables recipients to verify identity

3. Import a contact's public key:
 - Open a signed email from them > Security tab > Import their key
 - Or: Tools > OpenPGP Key Manager > Import from file

4. Send an encrypted email:
 - Compose a message to a recipient whose key you have
 - Click the lock icon in the toolbar
 - If their key is available, "Encrypt" option becomes active
 - Send. Thunderbird encrypts automatically

5. Receive and decrypt:
 - Thunderbird decrypts automatically when you open the message
 - Displays "This message was encrypted and signed by..."

Step 10 - Sign-Only Mode (For Authenticity Without Encryption)

Signing without encryption lets anyone verify the message came from you, without needing to exchange keys first:

```bash
gpg --clearsign message.txt
Creates message.txt.asc with readable text + signature block
```

The recipient can verify:

```bash
gpg --verify message.txt.asc
```

This is useful for software releases, announcements, and situations where confidentiality isn't the goal but authenticity is.

Step 11 - Web of Trust and Key Signing

The web of trust is a decentralized way to verify that a key actually belongs to the person it claims to. When you sign someone's key, you're vouching that you've verified the key belongs to them.

```bash
Sign someone's key (only after verifying fingerprint in person or via secure channel)
gpg --sign-key recipient@example.com

Send the signed key back to the key server
gpg --keyserver keys.openpgp.org --send-keys RECIPIENT_KEY_ID
```

In practice, most people just verify fingerprints out-of-band rather than building formal trust chains.

Step 12 - Key Management Commands

```bash
List all keys in your keyring
gpg --list-keys

Update keys from key server (check for revocations)
gpg --refresh-keys

Extend key expiration
gpg --edit-key you@example.com
At gpg> prompt:
  expire       (to change primary key expiry)
  save

Revoke a compromised key
gpg --import yourname-revoke.asc
gpg --keyserver keys.openpgp.org --send-keys YOUR_KEY_ID
```

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

Can I trust these tools with sensitive data?

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

Related Articles

- [Best Email Encryption Plugin Thunderbird](/best-email-encryption-plugin-thunderbird/)
- [How to Set Up S/MIME Email Encryption: A Practical Guide](/how-to-set-up-smime-email-encryption/)
- [Use GPG Signed Emails to Verify Sender Identity](/how-to-use-gpg-signed-emails-to-verify-sender-identity/)
- [How to Use GPG Signed Emails to Verify Sender Identity](/how-to-use-gpg-signed-emails-to-verify-sender-identity/)
- [Secure Email Forwarding With Encryption How To Set Up](/secure-email-forwarding-with-encryption-how-to-set-up-anonad/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
