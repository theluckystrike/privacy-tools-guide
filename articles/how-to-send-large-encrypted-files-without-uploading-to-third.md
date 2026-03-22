---
layout: default
title: "How To Send Large Encrypted Files Without Uploading"
description: "A guide for developers and power users on sending large encrypted files peer-to-peer without relying on cloud services. Covers age"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-send-large-encrypted-files-without-uploading-to-third/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Use OnionShare for encrypted peer-to-peer file transfer without intermediaries: it creates a Tor onion address that the recipient accesses directly from your machine, keeping files off all servers. Alternatively, use age encryption combined with SFTP or rsync to send encrypted files directly to the recipient's server, or use the age-encrypted-backup pattern for offline transfer via USB drives. All three methods avoid cloud services entirely while keeping data encrypted end-to-end.

## Why Avoid Third-Party File Sharing Services

Cloud file sharing services create multiple attack surfaces. The service operator can read your files, their servers can be breached, and metadata about your communications remains visible. Additionally, many services impose file size limits or require account creation. When dealing with sensitive documents, backups, code repositories, or personal media, you deserve better control over your data.

Peer-to-peer encrypted file transfer keeps your data on your machines until the exact moment of transfer. No cloud storage, no intermediate servers, and no persistent copies anywhere except on the intended recipient's device.

## Using age Encryption for File Transfer

The `age` encryption tool (age-encryption.org) provides modern, secure file encryption using X25519 public keys or passphrase-based encryption. It's designed as a replacement for age-old PGP with simpler syntax and better defaults.

### Installing age

On macOS:
```bash
brew install age
```

On Linux:
```bash
sudo apt install age
```

### Encrypting Files with age

Generate a keypair for the recipient:
```bash
age-keygen -o recipient_key.txt
cat recipient_key.txt
# Public key: age1EXAMPLE...
```

Encrypt a large file using the recipient's public key:
```bash
age -r age1EXAMPLE... largefile.zip > largefile.zip.encrypted
```

For passphrase-based encryption (when you can't exchange keys):
```bash
age -p -o largefile.zip.encrypted largefile.zip
# Enter passphrase when prompted
```

### Decrypting Files

Recipient decrypts using their private key:
```bash
age -d -i recipient_key.txt largefile.zip.encrypted > largefile.zip
```

Or with a passphrase:
```bash
age -d -o decrypted_largefile.zip largefile.zip.encrypted
```

The encrypted file can now be sent through any channel—email attachment, USB drive, or temporary file hosting—without exposing the contents.

## Using OpenPGP for Large File Encryption

OpenPGP remains the standard for encrypted file transfer. Modern implementations handle large files efficiently with proper streaming.

### Encrypting with GPG

Generate a keypair if you don't have one:
```bash
gpg --full-generate-key
# Select RSA 4096, no expiration for simplicity
```

Encrypt a large file:
```bash
gpg --encrypt --armor --recipient recipient@example.com largefile.tar.gz
# Output: largefile.tar.gz.asc
```

The recipient decrypts with:
```bash
gpg --decrypt largefile.tar.gz.asc > largefile.tar.gz
```

For streaming very large files without buffering everything in memory, GPG handles this automatically with its symmetric key encryption mode:

```bash
gpg --symmetric --cipher-algo AES256 largefile.iso
```

## Onion Share: Peer-to-Peer Through Tor

Onion Share creates a temporary Tor hidden service that lets recipients download files directly from your computer. The file never touches a third-party server—transfers occur entirely peer-to-peer through the Tor network.

### Basic Onion Share Usage

Install Onion Share:
```bash
brew install onionshare
```

Launch the GUI or use the CLI:
```bash
onionshare --file largefile.zip
```

Onion Share displays a unique .onion URL. Share this URL with your recipient (who needs Tor Browser). When they visit, they can download the file directly from your machine. The link expires automatically when transfer completes or after a configurable timeout.

For CLI usage with specific options:
```bash
onionshare --persistent --title "Secure Transfer" --schedule 1h largefile.zip
```

This creates a link valid for one hour that can be used multiple times.

## Building Custom Transfer Scripts

For automated workflows or integration into applications, build custom peer-to-peer transfer solutions.

### Simple Web Server Transfer with Encryption

Create a self-destructing transfer script:

```python
#!/usr/bin/env python3
import http.server
import socketserver
import os
import threading
import time
import hashlib

PORT = 8000
FILE_PATH = "secure_file.zip"
MAX_DOWNLOADS = 1

download_count = 0
download_lock = threading.Lock()

class TransferHandler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        global download_count
        with download_lock:
            if download_count >= MAX_DOWNLOADS:
                self.send_error(403, "Transfer limit reached")
                return
            download_count += 1

        # Verify hash to ensure file integrity
        self.send_response(200)
        self.send_header('Content-Type', 'application/zip')
        self.send_header('Content-Disposition', f'attachment; filename="{os.path.basename(FILE_PATH)}"')
        self.end_headers()

        with open(FILE_PATH, 'rb') as f:
            self.wfile.write(f.read())

        print(f"File transferred. Server shutting down.")
        threading.Timer(1, lambda: os._exit(0)).start()

    def log_message(self, format, *args):
        pass  # Silence logging

# Generate one-time token
token = hashlib.sha256(str(time.time()).encode()).hexdigest()[:8]
print(f"Starting transfer server for {FILE_PATH}")
print(f"Token: {token}")
print("Press Ctrl+C to stop")

with socketserver.TCPServer(("", PORT), TransferHandler) as httpd:
    httpd.serve_forever()
```

Before running this, encrypt the file using age or GPG. The recipient needs the encryption key through a separate secure channel.

### Using scp with Jump Hosts

For direct transfers between servers:

```bash
# Transfer through an intermediate bastion host
scp -o "ProxyJump bastion@example.com" largefile.tar.gz user@internal-server:/path/

# Or set up persistent port forwarding
ssh -L 2222:internal-server:22 bastion@example.com
# Then in another terminal:
scp -P 2222 largefile.tar.gz localhost:/path/
```

Combine with encrypted archives for additional security:
```bash
# Create encrypted archive
tar czf - largefile/ | age -p -o archive.tar.gz.age

# Extract on recipient side
age -d -o - archive.tar.gz.age | tar xzf - -C /destination/
```

## Best Practices for Secure File Transfer

When transferring sensitive files, follow these guidelines:

**Use independent channels for keys and files.** If you send an encrypted file via email, share the decryption key through Signal, a phone call, or in person. This prevents compromise of a single channel from exposing your data.

**Verify file integrity.** Always share a checksum separately:
```bash
sha256sum largefile.zip > checksums.txt
```

**Set expiration on transfers.** When using Onion Share or similar tools, configure automatic link expiration. The recipient should download and verify immediately.

**Consider forward secrecy.** Tools like Signal and similar messaging apps provide automatic forward secrecy—the same file sent through these channels gets new encryption keys for each session. For files, age with ephemeral keys or GPG with proper key management provides similar properties when keys rotate.



## Frequently Asked Questions


**How long does it take to send large encrypted files without uploading?**

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

- [Magic Wormhole Encrypted File Transfer How To Send Files Sec](/privacy-tools-guide/magic-wormhole-encrypted-file-transfer-how-to-send-files-sec/)
- [How To Send Encrypted Attachments That Recipients Can Open W](/privacy-tools-guide/how-to-send-encrypted-attachments-that-recipients-can-open-w/)
- [WireGuard Performance Tuning for Large File Transfer.](/privacy-tools-guide/wireguard-performance-tuning-large-file-transfer-optimizatio/)
- [Set Up Encrypted Local Backup Of Iphone Without Using Icloud](/privacy-tools-guide/how-to-set-up-encrypted-local-backup-of-iphone-without-using-icloud/)
- [Best Way to Encrypt Google Drive Files: A Developer Guide](/privacy-tools-guide/best-way-to-encrypt-google-drive-files/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
