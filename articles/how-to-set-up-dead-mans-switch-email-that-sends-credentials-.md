---
layout: default
title: "Set Up a Dead Man's Switch Email That Sends Credentials If You Stop Checking In"
description: "A practical guide for developers and power users to create an automated system that delivers your credentials to trusted contacts if you become inactive."
date: 2026-03-16
author: "Privacy Tools Guide"
permalink: /how-to-set-up-dead-mans-switch-email-that-sends-credentials-/
reviewed: true
score: 8
voice-checked: true
categories: [guides]
intent-checked: true
---


{% raw %}

A dead man's switch is a safety mechanism that triggers a predefined action when the operator fails to perform a periodic check-in. In the context of digital security, this translates to an automated system that monitors your activity and sends sensitive information—such as passwords, encryption keys, or recovery phrases—to trusted contacts if you become unresponsive.

This guide walks you through building a credential-dispensing dead man's switch using cron jobs, email automation, and encrypted payloads. The solution targets developers and power users comfortable with command-line tools and basic scripting.

## Why Build Your Own Dead Man's Switch?

Commercial password managers offer inheritance features, but they come with limitations. You may want:

- **Full control** over when and how your credentials are released
- **Custom triggers** based on multiple check-in methods
- **No vendor lock-in**—your system works independently of any service
- **Encrypted delivery** that only your intended recipient can decrypt

Building your own switch gives you transparency and flexibility that third-party solutions cannot match.

## Core Architecture

The system consists of three components:

1. **Check-in script**—resets the inactivity timer
2. **Monitor script**—runs on a schedule and checks if you've exceeded the inactivity threshold
3. **Delivery mechanism**—sends encrypted credentials to your designated contact

You host these scripts on an always-online machine: a VPS, a home server, or even a Raspberry Pi connected to reliable power and internet.

## Step 1: Prepare Encrypted Credential Payload

Never store plaintext credentials in your scripts. Instead, create an encrypted file that only your recipient can decrypt.

Generate a GPG key pair for your recipient, or use a pre-existing public key. Then encrypt your credential file:

```bash
gpg --encrypt --recipient your-recipient@example.com \
  --output credentials.gpg credentials.txt
```

Store `credentials.gpg` in a secure location on your server. The plaintext `credentials.txt` should be deleted after encryption.

Your credential file might contain:

```
=== Personal Accounts ===
Email: your-email@example.com
Password: (use a password manager to generate)

=== Server Access ===
SSH Key: /home/user/.ssh/id_ed25519
VPS Root Password: (generated password)

=== Recovery Phrases ===
Bitcoin Wallet: (your 12-word seed)
Trezor PIN: (4-digit PIN)
```

## Step 2: Create the Check-in Script

The check-in script updates a timestamp file. When you run it, it signals that you are still active.

Create a file called `checkin.sh`:

```bash
#!/bin/bash
TIMESTAMP_FILE="/home/user/deadmanswitch/last_checkin"

# Update the timestamp to current time
date +%s > "$TIMESTAMP_FILE"

echo "Check-in recorded at $(date)"
```

Make it executable:

```bash
chmod +x checkin.sh
```

Run this script manually (or automate it) whenever you want to reset the dead man's switch. Many users run it from their phone using Termux, or from their desktop on a weekly basis.

## Step 3: Create the Monitor Script

The monitor script runs on a schedule (via cron) and compares the current time against the last check-in timestamp. If the difference exceeds your threshold, it triggers the delivery.

Create `monitor.sh`:

```bash
#!/bin/bash

TIMESTAMP_FILE="/home/user/deadmanswitch/last_checkin"
CREDENTIALS_FILE="/home/user/deadmanswitch/credentials.gpg"
RECIPIENT="your-trusted-contact@example.com"
INACTIVITY_DAYS=7

# Exit if no check-in file exists yet
if [ ! -f "$TIMESTAMP_FILE" ]; then
    exit 0
fi

# Calculate days since last check-in
LAST_CHECKIN=$(cat "$TIMESTAMP_FILE")
NOW=$(date +%s)
DIFF_SECONDS=$((NOW - LAST_CHECKIN))
DIFF_DAYS=$((DIFF_SECONDS / 86400))

if [ "$DIFF_DAYS" -ge "$INACTIVITY_DAYS" ]; then
    # Send the encrypted credentials
    mutt -s "Dead Man's Switch Triggered - Credentials" \
         -a "$CREDENTIALS_FILE" -- "$RECIPIENT" << EOF

Your contact has been inactive for $DIFF_DAYS days. 
The attached file contains their encrypted credentials.
Use their GPG key to decrypt: gpg --decrypt credentials.gpg

This is an automated message from their dead man's switch system.
EOF
    
    echo "Dead man's switch triggered - credentials sent at $(date)"
fi
```

This script uses `mutt` for email sending. Install it via your package manager:

```bash
# Debian/Ubuntu
sudo apt install mutt

# macOS
brew install mutt
```

Configure mutt with your SMTP settings or use a local mail transfer agent.

## Step 4: Set Up Cron Jobs

Schedule the monitor script to run daily:

```bash
crontab -e
```

Add this line:

```
0 9 * * * /home/user/deadmanswitch/monitor.sh
```

This runs the check every day at 9:00 AM. Adjust the timing based on your needs.

## Step 5: Add Redundancy with Multiple Check-in Methods

A single check-in method creates a single point of failure. Consider adding:

- **HTTP endpoint**: Host a simple web page that accepts a POST request to update the timestamp
- **GitHub gist**: Use a private gist that you update periodically—the modified date serves as your check-in
- **Telegram bot**: Create a simple bot that responds to a specific command and updates the timestamp

Here's an example of an HTTP-based check-in using a minimal Python Flask app:

```python
from flask import Flask, request
import os
from datetime import datetime

app = Flask(__name__)
TIMESTAMP_FILE = '/home/user/deadmanswitch/last_checkin'

@app.route('/checkin', methods=['POST'])
def checkin():
    token = request.headers.get('X-Checkin-Token')
    if token != os.environ.get('CHECKIN_TOKEN'):
        return 'Unauthorized', 401
    
    with open(TIMESTAMP_FILE, 'w') as f:
        f.write(str(int(datetime.now().timestamp())))
    
    return 'OK', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Run this alongside your monitor script. To check in, send:

```bash
curl -X POST https://your-server.example.com/checkin \
  -H "X-Checkin-Token: your-secret-token"
```

## Security Considerations

- **Store credentials securely**: Keep the encrypted file on an encrypted filesystem or hardware token
- **Limit script permissions**: Run scripts with the minimum necessary privileges
- **Use two-factor authentication**: Enable 2FA on the email account used for delivery
- **Test your system**: Run a dry test where you simulate inactivity and verify the email sends correctly
- **Rotate credentials periodically**: Update the encrypted payload when you change passwords

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
