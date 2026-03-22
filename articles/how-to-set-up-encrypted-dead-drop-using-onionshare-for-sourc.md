---
layout: default
title: "How To Set Up Encrypted Dead Drop Using Onionshare"
description: "Build an anonymous file drop box with OnionShare over Tor. Step-by-step setup for journalists and whistleblowers needing secure submissions."
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-encrypted-dead-drop-using-onionshare-for-sourc/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---
---
layout: default
title: "How To Set Up Encrypted Dead Drop Using Onionshare"
description: "A practical guide for developers and power users to create secure, anonymous drop boxes using OnionShare"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-encrypted-dead-drop-using-onionshare-for-sourc/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true---

Set up an OnionShare dead drop by installing the application, creating a receive-only mode, and hosting it as a Tor onion service that accepts file uploads over encrypted channels. Sources access the service via an one-time URL over Tor, submit documents, and disconnect without leaving identifying information. OnionShare never logs IP addresses, doesn't require accounts, and automatically deletes files after retrieval—creating truly anonymous submission channels for developers and power users protecting sources.

## Understanding the Dead Drop Model

A dead drop works like a physical mailbox in espionage tradecraft: one party leaves information at a predetermined location for another party to retrieve later. In digital form, this means your source can submit encrypted documents or messages that only you can decrypt, without any direct communication or IP address correlation.

OnionShare provides the infrastructure by hosting a Tor onion service—a private, encrypted website accessible only through the Tor browser. The connection is end-to-end encrypted, and the server leaves no logs on traditional internet infrastructure.

The key advantage over alternatives like SecureDrop (the Freedom of the Press Foundation's journalism-grade submission system) is deployment simplicity. SecureDrop requires a dedicated air-gapped server and significant operational overhead. OnionShare runs on a standard laptop or VPS and can be operational in under ten minutes. For individuals, small publications, and researchers rather than large newsrooms, OnionShare hits a practical balance between security and setup burden.

## Prerequisites

Before setting up your dead drop, ensure you have:

- **OnionShare** (v2.6 or later) installed on your system
- **Tor Browser** for accessing onion services
- A persistent machine to run the OnionShare server (this can be a VPS or local device)
- GPG keys for encryption (optional but recommended for additional security)

For a persistent dead drop that remains available around the clock, a VPS is more reliable than a laptop that gets closed or rebooted. DigitalOcean's $6/month Droplet running Ubuntu 22.04 is sufficient. OnionShare supports headless operation through its CLI, making server deployment practical.

## Step-by-Step Setup

### 1. Install OnionShare

On macOS with Homebrew:

```bash
brew install --cask onionshare
```

On Linux:

```bash
sudo apt install onionshare
```

On Windows, download the installer from the official website.

For server deployment on a headless Linux VPS, install the CLI package:

```bash
sudo apt install onionshare-cli
```

The CLI version exposes all the same functionality without requiring a graphical desktop environment—essential for running the dead drop on a remote server.

### 2. Configure OnionShare for Dead Drop Mode

Launch OnionShare and select **"Receive Files"** mode. This configures OnionShare as a dead drop where sources can upload files to your server.

In the settings panel, configure these options:

- **Connection Timeout**: Set to 60 seconds to accommodate slower Tor connections
- **Auto-start Tor**: Enable this for easier operation
- **Disable close button**: Prevents accidental shutdown

For CLI operation on a server, the equivalent command is:

```bash
onionshare-cli --receive --persistent /path/to/config.json --no-autostop-sharing
```

The `--no-autostop-sharing` flag keeps the service running indefinitely rather than shutting down after the first upload—critical for an always-on dead drop.

### 3. Set Up Receive Options

Configure the receive behavior in the settings:

```text
Maximum file size: 50MB (adjust based on your needs)
Receive length: 0 (unlimited—sources can submit anytime)
```

You can also add a custom welcome message that sources see when they visit your onion service. Keep this message minimal—instructions to use Tor Browser, an assurance that no logs are kept, and a contact method for questions if needed.

### 4. Generate and Secure Your Onion Address

OnionShare generates a unique .onion URL. This address serves as your dead drop location. The application provides two versions:

- **Long address**: More secure, harder to brute-force
- **Short address**: Easier to share, slightly less secure

For a source dead drop, share the long address. Copy this address and store it securely—you cannot recover it if lost.

OnionShare v2.6 and later generates v3 onion addresses by default. These are 56-character addresses (rather than the 16-character v2 addresses) and offer significantly stronger cryptographic security. Verify that your generated address is 56 characters before distributing it to sources. Older versions of OnionShare or Tor may generate v2 addresses—update both to current versions.

### 5. Enable Persistent Storage

By default, received files are stored in OnionShare's temporary directory. Configure a persistent storage location:

1. Go to **Settings** → **Receive**
2. Set **"Save files to"** to a dedicated directory
3. Ensure proper filesystem permissions

```bash
mkdir -p ~/OnionShare/drops
chmod 700 ~/OnionShare/drops
```

On a VPS, consider encrypting the directory where incoming files land using `cryptsetup` or a LUKS-encrypted volume. This ensures that if the server is seized or imaged, the submitted files cannot be read without your encryption passphrase.

### 6. Add Encryption Layer (Optional but Recommended)

For additional security, encrypt files before processing them. Create a simple GPG wrapper:

```python
#!/usr/bin/env python3
import gnupg
import os
from pathlib import Path

class SecureDrop:
    def __init__(self, gpg_home='/path/to/gpg/home'):
        self.gpg = gnupg.GPG(gnupghome=gpg_home)

    def encrypt_file(self, filepath, recipient_keyid):
        with open(filepath, 'rb') as f:
            encrypted = self.gpg.encrypt_file(
                f,
                recipients=[recipient_keyid],
                output=f"{filepath}.gpg"
            )
        return encrypted.ok

# Usage
drop = SecureDrop()
drop.encrypt_file('uploaded_file.bin', 'YOUR_KEY_ID')
```

This ensures that even if someone compromises your server, they cannot read the submitted content without your private GPG key.

A practical workflow: configure a filesystem watcher (inotifywait on Linux) to trigger the GPG encryption script automatically whenever a new file appears in the OnionShare receive directory. This way, submitted files are encrypted within seconds of arrival, minimizing the window during which they exist in plaintext.

```bash
# Watch for new files and auto-encrypt
inotifywait -m ~/OnionShare/drops -e create |
while read path action file; do
    python3 /usr/local/bin/encrypt-drop.py "$path$file" YOUR_GPG_KEY_ID
    echo "Encrypted: $file at $(date)" >> ~/OnionShare/log.txt
done
```

## Operational Security Considerations

Running a dead drop requires attention to operational security:

### Network Isolation

Run OnionShare on an isolated network segment if possible. Consider using a dedicated VPN in addition to Tor to prevent traffic correlation attacks. However, be aware that VPNs introduce a different trust dependency—your VPN provider can see that you are using Tor, which in some threat models is undesirable. For most use cases, Tor alone provides sufficient anonymity for the server-side operator.

### File Handling

Create a processing workflow that minimizes exposure:

1. Download files from OnionShare to an air-gapped machine
2. Transfer to a secure analysis environment
3. Wipe the original uploads immediately

For transferring files from the server to an air-gapped workstation, use an encrypted USB drive. Mount the drive, copy the GPG-encrypted files, unmount, then shred the originals on the server:

```bash
shred -u ~/OnionShare/drops/original_file.bin
```

`shred` overwrites the file data before deletion, making recovery harder on traditional spinning disk drives. On SSDs with wear leveling, shred is less effective—full disk encryption at the volume level is the more reliable protection on SSDs.

### Metadata Stripping

Sources should strip metadata from documents before submission. Provide them with tools or instructions:

```bash
# Using exiftool to strip metadata
exiftool -all= document.pdf
exiftool -all= image.jpg
```

Consider including a note in your welcome message directing sources to the MAT2 (Metadata Anonymisation Toolkit) tool, which handles a broader range of file formats than exiftool and is available in the Tails OS default installation. Sources using Tails for submission already benefit from Tor integration at the OS level.

### Source Instructions

The instructions you provide to sources are part of your security model. Poorly written instructions create risk if sources misunderstand the process. A clear template:

```
To submit information securely:

1. Download Tor Browser from torproject.org
2. Copy and paste this address into Tor Browser:
   [your-onion-address].onion

3. Follow the on-screen instructions to upload files
4. Use MAT2 or exiftool to strip metadata from documents before uploading

Important: Close Tor Browser after submission to protect your session.
Do not reuse this session for other browsing.
```

## Sharing the Dead Drop Address

When providing the onion address to sources, use multiple channels:

- Split the URL across different communication platforms
- Use a secure messenger with disappearing messages
- Include instructions for accessing via Tor Browser

For high-stakes contexts, consider publishing the onion address in a verifiable public location—your organization's website, a signed keybase post, or a PGP-signed message—so sources can independently verify they have the correct address and not one planted by an adversary.

## Troubleshooting Common Issues

### Connection Problems

If sources cannot connect to your dead drop:

- Verify OnionShare is running and the Tor connection is active
- Check that port 80 is not blocked by your firewall
- Ensure the onion service has fully initialized (wait 30-60 seconds after startup)

Check OnionShare's status from the CLI:

```bash
onionshare-cli --receive --verbose
```

The verbose flag shows Tor circuit establishment progress and will indicate if the onion service descriptor has been published to the Tor network.

### Large File Uploads

OnionShare may time out with large files. Instruct sources to:
- Split large files into smaller chunks (under 10MB each)
- Use compression before upload
- Retry if the first attempt fails

For archives, 7-Zip with AES-256 encryption adds an additional layer before transit—even if the Tor connection is compromised, the archive content remains protected.

### Server Availability

For high-stakes operations, run OnionShare on a VPS with uptime guarantees. Configure automatic restart scripts:

```bash
#!/bin/bash
while true; do
    onionshare --receive --persistent ~/OnionShare/config.json
    sleep 5
done
```

A systemd service unit is more durable than a shell loop for production deployments:

```ini
[Unit]
Description=OnionShare Dead Drop
After=network.target tor.service

[Service]
User=onionshare
ExecStart=/usr/bin/onionshare-cli --receive --persistent /home/onionshare/config.json --no-autostop-sharing
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable with `systemctl enable --now onionshare-drop`. The service restarts automatically on failure and starts at boot, ensuring availability even after VPS reboots or transient errors.

## Frequently Asked Questions

**How long does it take to set up encrypted dead drop using onionshare?**

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

- [How to Set Up Secure Dead Drop for Digital Information](/privacy-tools-guide/how-to-set-up-secure-dead-drop-for-digital-information/)
- [Set Up Dead Man's Switch Using Cron Job to Release Encrypted](/privacy-tools-guide/how-to-set-up-dead-mans-switch-using-cron-job-to-release-enc/)
- [Set Up a Dead Man's Switch Email That Sends Credentials If](/privacy-tools-guide/how-to-set-up-dead-mans-switch-email-that-sends-credentials-/)
- [Onionshare Secure File Sharing Over Tor Network Setup And Us](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Business Email Privacy: How to Set Up Encrypted Email.](/privacy-tools-guide/business-email-privacy-how-to-set-up-encrypted-email-for-com/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
