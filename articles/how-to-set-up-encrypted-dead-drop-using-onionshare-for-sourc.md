---
layout: default
title: "How to Set Up an Encrypted Dead Drop Using OnionShare."
description: "A practical guide for developers and power users to create secure, anonymous drop boxes using OnionShare."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-encrypted-dead-drop-using-onionshare-for-sourc/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

Set up an OnionShare dead drop by installing the application, creating a receive-only mode, and hosting it as a Tor onion service that accepts file uploads over encrypted channels. Sources access the service via a one-time URL over Tor, submit documents, and disconnect without leaving identifying information. OnionShare never logs IP addresses, doesn't require accounts, and automatically deletes files after retrieval—creating truly anonymous submission channels for developers and power users protecting sources.

## Understanding the Dead Drop Model

A dead drop works like a physical mailbox in espionage tradecraft: one party leaves information at a predetermined location for another party to retrieve later. In digital form, this means your source can submit encrypted documents or messages that only you can decrypt, without any direct communication or IP address correlation.

OnionShare provides the infrastructure by hosting a Tor onion service—a private, encrypted website accessible only through the Tor browser. The connection is end-to-end encrypted, and the server leaves no logs on traditional internet infrastructure.

## Prerequisites

Before setting up your dead drop, ensure you have:

- **OnionShare** (v2.6 or later) installed on your system
- **Tor Browser** for accessing onion services
- A persistent machine to run the OnionShare server (this can be a VPS or local device)
- GPG keys for encryption (optional but recommended for additional security)

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

### 2. Configure OnionShare for Dead Drop Mode

Launch OnionShare and select **"Receive Files"** mode. This configures OnionShare as a dead drop where sources can upload files to your server.

In the settings panel, configure these options:

- **Connection Timeout**: Set to 60 seconds to accommodate slower Tor connections
- **Auto-start Tor**: Enable this for easier operation
- **Disable close button**: Prevents accidental shutdown

### 3. Set Up Receive Options

Configure the receive behavior in the settings:

```text
Maximum file size: 50MB (adjust based on your needs)
Receive length: 0 (unlimited—sources can submit anytime)
```

You can also add a custom welcome message that sources see when they visit your onion service.

### 4. Generate and Secure Your Onion Address

OnionShare generates a unique .onion URL. This address serves as your dead drop location. The application provides two versions:

- **Long address**: More secure, harder to brute-force
- **Short address**: Easier to share, slightly less secure

For a source dead drop, share the long address. Copy this address and store it securely—you cannot recover it if lost.

### 5. Enable Persistent Storage

By default, received files are stored in OnionShare's temporary directory. Configure a persistent storage location:

1. Go to **Settings** → **Receive**
2. Set **"Save files to"** to a dedicated directory
3. Ensure proper filesystem permissions

```bash
mkdir -p ~/OnionShare/drops
chmod 700 ~/OnionShare/drops
```

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

## Operational Security Considerations

Running a dead drop requires attention to operational security:

### Network Isolation

Run OnionShare on an isolated network segment if possible. Consider using a dedicated VPN in addition to Tor to prevent traffic correlation attacks.

### File Handling

Create a processing workflow that minimizes exposure:

1. Download files from OnionShare to an air-gapped machine
2. Transfer to a secure analysis environment
3. Wipe the original uploads immediately

### Metadata Stripping

Sources should strip metadata from documents before submission. Provide them with tools or instructions:

```bash
# Using exiftool to strip metadata
exiftool -all= document.pdf
exiftool -all= image.jpg
```

## Sharing the Dead Drop Address

When providing the onion address to sources, use multiple channels:

- Split the URL across different communication platforms
- Use a secure messenger with disappearing messages
- Include instructions for accessing via Tor Browser

Example formatted instructions:

```
To submit information securely:

1. Download Tor Browser from torproject.org
2. Copy and paste this address into Tor Browser:
   [your-onion-address].onion

3. Follow the on-screen instructions to upload files

Important: Close Tor Browser after submission to protect your session.
```

## Troubleshooting Common Issues

### Connection Problems

If sources cannot connect to your dead drop:

- Verify OnionShare is running and the Tor connection is active
- Check that port 80 is not blocked by your firewall
- Ensure the onion service has fully initialized (wait 30-60 seconds after startup)

### Large File Uploads

OnionShare may time out with large files. Instruct sources to:
- Split large files into smaller chunks (under 10MB each)
- Use compression before upload
- Retry if the first attempt fails

### Server Availability

For high-stakes operations, run OnionShare on a VPS with uptime guarantees. Configure automatic restart scripts:

```bash
#!/bin/bash
while true; do
    onionshare --receive --persistent ~/OnionShare/config.json
    sleep 5
done
```

## Conclusion

OnionShare provides a robust foundation for secure source communication. By combining its built-in encryption with additional layers like GPG and proper operational security practices, you can create a dead drop that protects both your sources and your identity.

Remember that security is a chain—the strongest link determines overall protection. Invest time in securing every component of your drop infrastructure, and your sources will benefit from genuinely anonymous submission channels.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
