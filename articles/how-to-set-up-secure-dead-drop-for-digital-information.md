---
layout: default
title: "How to Set Up Secure Dead Drop for Digital Information"
description: "A practical guide to building secure digital dead drops for anonymous information exchange. Learn self-hosted solutions, PGP-based methods, and automation for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-secure-dead-drop-for-digital-information/
categories: [guides]
intent-checked: true
voice-checked: true
---

{% raw %}

A digital dead drop works like its physical counterpart: one person leaves information in a predetermined location for another person to retrieve later, without both parties needing to meet. In an era of mass surveillance and data harvesting, understanding how to set up secure dead drops for digital information provides a valuable tool for privacy-conscious developers, security researchers, whistleblowers, and anyone needing to exchange sensitive information without exposing their identities or communication patterns.

This guide covers multiple approaches to building secure dead drops, from simple PGP-based message exchange to self-hosted web applications. Each method offers different tradeoffs between convenience, anonymity, and security.

## Understanding Dead Drop Security Requirements

Before implementing a dead drop system, you need to establish your threat model. A secure digital dead drop should provide:

- **Sender anonymity**: The recipient cannot determine who uploaded the message
- **Recipient anonymity**: The sender does not know who retrieved the message (for reply drops)
- **Message confidentiality**: Only the intended recipient can read the message
- **Metadata protection**: Timing patterns and access logs should not reveal participant identities
- **Expiration**: Messages should auto-delete after retrieval or a set time

No single solution provides perfect anonymity, but layering these properties creates robust communication channels.

## Method 1: PGP-Based Dead Drop with Pastebin

The simplest approach uses GPG encryption combined with a temporary paste service. This requires no special software and works across any platform with GPG installed.

First, generate a dedicated dead drop keypair for the recipient:

```bash
gpg --full-generate-key
# Select RSA, 4096 bits, no expiration
# Use a dedicated email like deaddrop@example.com
```

Export the public key and share it securely (in person or via a separate channel):

```bash
gpg --export --armor deaddrop@example.com > deaddrop-public.key
```

Senders encrypt their message using this public key:

```bash
echo "Secret message for the dead drop" | gpg \
  --encrypt \
  --armor \
  --recipient deaddrop@example.com \
  --output encrypted-message.asc
```

The encrypted message can now be posted to any pastebin service. The recipient downloads, decrypts, and reads:

```bash
gpg --decrypt encrypted-message.asc
```

For reply functionality, the recipient creates their own keypair and publishes the public key. The sender retrieves it, encrypts their response, and posts it back to the dead drop.

**Limitations**: This method relies on third-party paste services, which may log IP addresses. Timing analysis can correlate upload and download events.

## Method 2: Self-Hosted PrivateBin

PrivateBin provides a self-hosted pastebin with client-side encryption. The server never sees plaintext content, making it suitable for dead drops.

### Installing PrivateBin

PrivateBin requires PHP and a web server. Using Docker simplifies deployment:

```bash
docker run -d --name privatebin \
  -p 8080:8080 \
  -v privatebin-data:/data \
  -e PASSWORDPHRASE=changeme \
  privatebin/nginx:latest
```

### Configuration for Dead Drop Use

Edit the configuration file at `/var/www/privatebin/cfg/conf.php`:

```php
<?php
$cfg = [
    // Enable burn after reading (one-time view)
    'burnafter' => true,
    
    // Expire discussions after 1 hour
    'defaultexpiration' => '1hour',
    
    // Disable IP logging
    'ip' => false,
    
    // Require password for creation
    'password' => true,
    
    // Enable discussion (replies)
    'discussion' => true,
    
    // Open discussion means anyone with the link can reply
    'opendiscussion' => false,
];
```

### Workflow

1. Sender visits the PrivateBin URL
2. Enters the dead drop password (shared separately)
3. Types message, enables "burn after reading"
4. Shares the generated URL with recipient
5. Recipient opens URL once, then message self-destructs

For bidirectional communication, both parties exchange URLs through the drop.

## Method 3: OnionShare for Persistent Dead Drops

OnionShare creates temporary Tor hidden services for file transfer. It works without a dedicated server and provides strong anonymity through Tor's onion routing.

### Installing OnionShare

```bash
# macOS
brew install onionshare

# Linux
sudo apt install onionshare

# Or use the AppImage
wget https://github.com/micahflee/onionshare/releases/latest/download/OnionShare-x86_64.AppImage
chmod +x OnionShare-x86_64.AppImage
```

### Creating a Persistent Dead Drop

Launch OnionShare and configure a static, persistent onion address:

```bash
onionshare --persistent --address mydeaddrop.onion \
  --data-dir /var/deaddrop/data \
  --verbose
```

This creates a permanent `.onion` URL. The first time you run it, OnionShare generates a private key stored locally—backup this key to maintain the same address across reinstalls.

### Automating Dead Drop Access

For programmatic access, use the OnionShare CLI with the receive mode:

```bash
onionshare --receive \
  --persistent \
  --data-dir /var/deaddrop/incoming \
  --title "Secure Dead Drop" \
  --verbose
```

The receive mode creates a web interface where files are uploaded to a designated folder. Configure the web server to auto-delete files after retrieval using a cron job:

```bash
# Delete files older than 24 hours
0 */6 * * * find /var/deaddrop/incoming -type f -mmin +1440 -delete
```

## Method 4: Custom Dead Drop API with Encryption

For developers wanting full control, build a minimal dead drop API with explicit encryption handling.

### Python Flask Implementation

```python
from flask import Flask, request, jsonify, send_from_directory
import os
import uuid
import time
from cryptography.fernet import Fernet

app = Flask(__name__)
DROP_DIR = '/var/deaddrop/messages'
EXPIRATION_SECONDS = 3600

# Generate or load encryption key
KEY_FILE = '/var/deaddrop/.key'
if os.path.exists(KEY_FILE):
    with open(KEY_FILE, 'rb') as f:
        KEY = f.read()
else:
    KEY = Fernet.generate_key()
    with open(KEY_FILE, 'wb') as f:
        f.write(KEY)

cipher = Fernet(KEY)
os.makedirs(DROP_DIR, exist_ok=True)

@app.route('/api/deaddrop', methods=['POST'])
def create_message():
    """Create a new encrypted dead drop message."""
    data = request.json
    encrypted_message = data.get('message')
    
    if not encrypted_message:
        return jsonify({'error': 'Message required'}), 400
    
    message_id = str(uuid.uuid4())
    expiry = time.time() + EXPIRATION_SECONDS
    
    # Store encrypted message with expiry timestamp
    filepath = os.path.join(DROP_DIR, f"{message_id}.enc")
    with open(filepath, 'w') as f:
        f.write(f"{expiry}\n{encrypted_message}")
    
    return jsonify({
        'id': message_id,
        'expires': expiry
    })

@app.route('/api/deaddrop/<message_id>', methods=['GET'])
def get_message(message_id):
    """Retrieve and delete a message (one-time access)."""
    filepath = os.path.join(DROP_DIR, f"{message_id}.enc")
    
    if not os.path.exists(filepath):
        return jsonify({'error': 'Not found'}), 404
    
    with open(filepath, 'r') as f:
        lines = f.readlines()
    
    expiry = float(lines[0])
    if time.time() > expiry:
        os.remove(filepath)
        return jsonify({'error': 'Expired'}), 410
    
    # Delete after retrieval (burn after reading)
    os.remove(filepath)
    
    return jsonify({'message': lines[1].strip()})

@app.route('/api/cleanup', methods=['POST'])
def cleanup():
    """Remove expired messages."""
    count = 0
    for filename in os.listdir(DROP_DIR):
        filepath = os.path.join(DROP_DIR, filename)
        if os.path.isfile(filepath):
            with open(filepath, 'r') as f:
                expiry = float(f.readline())
            if time.time() > expiry:
                os.remove(filepath)
                count += 1
    return jsonify({'deleted': count})

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

**Security notes**: 
- Run behind Tor for anonymous hosting
- Use HTTPS in production
- Consider adding rate limiting to prevent enumeration
- Implement authentication for management endpoints

## Security Considerations

Regardless of which method you choose, follow these best practices:

1. **Use Tor Browser** for all dead drop interactions to prevent IP leakage
2. **Verify public keys** through separate channels before sending sensitive information
3. **Use unique passwords** for each dead drop service
4. **Enable auto-expiration** on all messages
5. **Regularly clear browser history** and cookies after accessing dead drops
6. **Consider timing**—avoid accessing dead drops at predictable intervals

For highest-security scenarios, combine methods: encrypt with PGP before uploading to a self-hosted service accessible only through Tor.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
