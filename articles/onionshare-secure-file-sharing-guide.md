---
layout: default
title: "Secure File Sharing with OnionShare"
description: "Use OnionShare to share files, host anonymous websites, and receive documents over Tor without exposing sender or recipient identities or IP addresses"
date: 2026-03-22
author: theluckystrike
permalink: /onionshare-secure-file-sharing-guide/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Secure File Sharing with OnionShare

OnionShare creates a temporary .onion address and runs a local web server behind it. The person you share with visits the .onion address in Tor Browser and downloads the file directly from your machine — no cloud storage intermediary, no account, no IP visible to either party. For receiving files (whistleblower drops, source submissions), it runs in receive mode — an upload form at a private .onion address.

---

## Install OnionShare

```bash
# Linux (Flatpak — recommended for most distributions)
flatpak install flathub org.onionshare.OnionShare

# Debian/Ubuntu
sudo apt install onionshare

# Fedora
sudo dnf install onionshare

# macOS
brew install --cask onionshare

# Windows
# Download installer from onionshare.org
# Verify GPG signature before running

# Check version
onionshare --version
```

OnionShare automatically downloads and manages its own Tor binary — you don't need Tor Browser or a separate Tor installation.

---

## Share Files (Send Mode)

### GUI Usage

1. Open OnionShare
2. **Share Files** tab
3. Drag files/folders into the window
4. Click **Start Sharing**
5. OnionShare generates a .onion address with a private key (2-word password)
6. Share the address + key with the recipient via Signal or another private channel
7. Recipient opens Tor Browser → visits the .onion URL → downloads

**Settings for sharing:**
- **Stop sharing after first download**: recommended for sensitive files — the server goes down after one person downloads
- **Public mode** (no password): for sharing publicly, not for private transfers

### CLI Usage

```bash
# Share a single file
onionshare-cli document.pdf

# Share a directory
onionshare-cli --zip ~/sensitive-folder/

# Share and stop after first download (most secure)
onionshare-cli --auto-stop-sharing document.pdf

# Share persistently (keep the same .onion address across restarts)
onionshare-cli --persistent ~/keys/my-persistence-key.json document.pdf

# Share without password (public)
onionshare-cli --no-auth document.pdf

# Output only the URL (for scripting)
onionshare-cli --quiet document.pdf
```

Example output:

```
OnionShare | https://onionshare.org/

Connecting to the Tor network: Done
Starting an onion service: Done

 ╭────────────────────────────────────────╮
 │ Give this address and private key to  │
 │ the person you're sharing with:       │
 │                                       │
 │ http://onionv3address.onion           │
 │ Private key: WORDONE-WORDTWO         │
 ╰────────────────────────────────────────╯

Waiting for a connection...
```

---

## Receive Files (Receive Mode)

Receive mode sets up an upload form at a .onion address. Sources can upload files to you without knowing who you are or where your server is.

```bash
# Start receive mode
onionshare-cli --receive

# Save received files to specific directory
onionshare-cli --receive --data-dir ~/received-files/

# Persistent receive address (same address each time — useful as a published drop)
onionshare-cli --receive --persistent ~/keys/receive-key.json
```

**For anonymous source drops:**

```
Share the .onion address + private key with sources
Post it on your website or send directly
Sources open Tor Browser → visit address → upload files
Files appear in your configured receive directory
You never learn the source's IP or identity
```

This is a lightweight alternative to SecureDrop for individual journalists or researchers who don't need a full SecureDrop installation.

---

## Host an Anonymous Website

OnionShare can host a static website as an .onion site — no domain registration, no hosting provider:

```bash
# Create your website files
mkdir ~/my-onion-site
echo "<h1>Anonymous site</h1>" > ~/my-onion-site/index.html

# Host the site
onionshare-cli --website ~/my-onion-site/

# For persistent address (same .onion every time you start)
onionshare-cli --website \
  --persistent ~/keys/website-key.json \
  ~/my-onion-site/
```

The site is accessible only via Tor Browser at the generated .onion address. No DNS, no registrar, no hosting account.

---

## Chat (Anonymous Chat Room)

OnionShare includes an ephemeral chat mode — no messages stored, no accounts:

```bash
# Start an anonymous chat room
onionshare-cli --chat

# Share the .onion URL with participants
# They open Tor Browser → enter the URL → pick a username → chat
# All messages are over Tor, no logs kept
```

This is useful for one-time secure group discussions where Signal or Matrix isn't appropriate.

---

## Persistent .onion Addresses

A persistent key means you can publish your .onion address and have it remain the same when you restart OnionShare:

```bash
# Generate and save a persistent key
onionshare-cli --receive \
  --persistent /home/user/.config/onionshare/dropbox-key.json

# The first run creates the key file
# Subsequent runs use the same .onion address
# Your published address works every time you run OnionShare

# Key file format (you can also create manually):
# Contains: private_key, public_key (ed25519 for v3 onions)
```

---

## Security Considerations

### What OnionShare Protects

- Your IP address is hidden from the recipient (Tor)
- Recipient's IP is hidden from you (Tor)
- File contents are encrypted in transit (Tor encryption)
- No cloud intermediary can see the files
- Transfer is direct: your machine → Tor → their machine

### What It Doesn't Protect

- **File metadata**: If you share a Word document or PDF with author metadata, that metadata reveals identity information. Strip metadata before sharing:

```bash
# Strip metadata from PDF
exiftool -all= document.pdf

# Strip from Office documents
# File → Properties → Inspect Document → Remove All (Word)
# Or use MAT2:
mat2 document.pdf
mat2 document.docx
```

- **Timing correlation**: If an adversary monitors when you start OnionShare and when a download occurs, they can potentially correlate. Use at random times, not immediately after receiving a request.

- **Your machine's security**: If your machine is compromised, OnionShare can't help. The adversary sees the files before they're shared.

### Tor Connectivity

If you're on a network that blocks Tor (corporate, school, some countries), configure bridges in OnionShare:

```
OnionShare Settings → Tor → Use Tor bridges
Choose: Obfs4 bridges (most effective for censored networks)
Paste custom bridges from bridges.torproject.org
```

---

## Verify Your .onion Address

Before sharing your address with someone, verify it's the one OnionShare is actually serving:

```bash
# Start OnionShare and note the generated address
onionshare-cli document.pdf

# In a separate Tor Browser session, visit the address
# Verify the download works before sharing widely

# For receive mode: test the upload form
# Visit your .onion in Tor Browser → try uploading a test file
# Verify it appears in your receive directory
```

---

## Scripting OnionShare for Automation

```python
#!/usr/bin/env python3
"""Automate OnionShare for scheduled anonymous file drops."""
import subprocess
import os
import sys
from pathlib import Path

def share_file_onionshare(filepath: str, auto_stop: bool = True) -> str:
    """Share a file via OnionShare and return the .onion URL."""
    cmd = ["onionshare-cli", "--quiet"]
    if auto_stop:
        cmd.append("--auto-stop-sharing")
    cmd.append(filepath)

    result = subprocess.run(cmd, capture_output=True, text=True)
    # Parse URL from output
    for line in result.stdout.splitlines():
        if ".onion" in line:
            return line.strip()
    raise RuntimeError(f"Could not get .onion URL: {result.stderr}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: share.py <file>")
        sys.exit(1)

    url = share_file_onionshare(sys.argv[1])
    print(f"Share this URL via Signal/Briar: {url}")
```

---

## Related Reading

- [OnionShare Secure File Sharing Over Tor Network](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [How to Set Up Encrypted Dead Drop with OnionShare](/privacy-tools-guide/how-to-set-up-encrypted-dead-drop-using-onionshare-for-sourc/)
- [Encrypted Messaging for Journalists Guide](/privacy-tools-guide/encrypted-messaging-for-journalists-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
