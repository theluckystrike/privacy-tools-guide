---
layout: default
title: "ProtonMail Import Export Tool Guide"
description: "Export Proton Mail data using three methods: the web interface (Settings > Download my data) for one-time MBOX exports, the IMAP Bridge (brew install --cask"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /protonmail-import-export-tool-guide/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---


{% raw %}
# ProtonMail Import Export Tool Guide: Complete Technical Reference

Export Proton Mail data using three methods: the web interface (Settings > Download my data) for one-time MBOX exports, the IMAP Bridge (`brew install --cask proton-mail-bridge`) for continuous desktop client sync, or the Proton Mail API for automated developer integrations. For imports, upload MBOX files through the web interface or push messages via IMAP. This guide covers step-by-step configuration for each method, including Python automation scripts and cron-based backup scheduling.

## Understanding ProtonMail's Data Export Options

Proton Mail provides several pathways for exporting data, each with different capabilities and use cases. The primary methods include the web interface export, IMAP Bridge for continuous sync, and API-based extraction for automated workflows.

The web interface allows you to export individual folders or your entire mailbox as MBOX format. This works well for one-time migrations but becomes tedious if you need regular automated backups. For developers building integration tools, the Proton Mail API provides programmatic access to emails, contacts, and calendar events.

## Exporting via Proton Mail Web Interface

The most straightforward export method uses the Proton Mail web application. Navigate to Settings → Download my data → Export. You can select specific folders or export everything including emails, contacts, and calendar entries.

The export generates a MBOX file that works with most email clients. For large mailboxes, expect the process to take several hours since Proton processes exports asynchronously for security reasons.

## Using Proton Mail IMAP Bridge

For continuous synchronization with desktop email clients, Proton Mail offers the IMAP Bridge application. This tool enables you to connect Thunderbird, Apple Mail, or any IMAP-compatible client to your Proton Mail account.

### Installing and Configuring IMAP Bridge

```bash
# macOS installation via Homebrew
brew install --cask proton-mail-bridge

# Debian/Ubuntu
sudo apt install protonmail-bridge
```

After installation, launch the bridge and authenticate with your Proton Mail credentials. The bridge provides separate IMAP and SMTP credentials specifically for external client connections. Configure your email client using these credentials:

```
IMAP Server: 127.0.0.1
IMAP Port: 1143
SMTP Server: 127.0.0.1
SMTP Port: 1025
```

### Exporting via IMAP with Python

Once IMAP Bridge runs, you can automate exports using Python's imaplib:

```python
import imaplib
import email
from datetime import datetime, timedelta

def connect_imap_bridge():
    """Connect to local Proton IMAP Bridge"""
    mail = imaplib.IMAP4('127.0.0.1', 1143)
    mail.login('your@protonmail.com', 'bridge_app_password')
    return mail

def export_folder_to_mbox(mail, folder_name, output_file):
    """Export a specific folder to MBOX format"""
    mail.select(folder_name)
    typ, messages = mail.search(None, 'ALL')
    message_ids = messages[0].split()
    
    with open(output_file, 'wb') as f:
        for msg_id in message_ids:
            typ, msg_data = mail.fetch(msg_id, '(RFC822)')
            for response_part in msg_data:
                if isinstance(response_part, tuple):
                    msg_content = response_part[1]
                    f.write(msg_content)
    
    print(f"Exported {len(message_ids)} messages to {output_file}")

# Usage example
mail = connect_imap_bridge()
export_folder_to_mbox(mail, 'Inbox', 'inbox_backup.mbox')
mail.logout()
```

## API-Based Export for Developers

Proton Mail's API offers the most flexibility for custom integrations. To use the API, you need to create an app in your Proton Mail account settings and obtain OAuth credentials.

### Setting Up API Access

```python
import requests
from requests_oauthlib import OAuth2Session

# Proton Mail API endpoints
AUTH_URL = 'https://api.protonmail.ch/oauth/authorize'
TOKEN_URL = 'https://api.protonmail.ch/oauth/token'

# Your app credentials
CLIENT_ID = 'your_client_id'
CLIENT_SECRET = 'your_client_secret'
REDIRECT_URI = 'http://localhost:8080/callback'

def get_oauth_token():
    """Obtain OAuth token for API access"""
    oauth = OAuth2Session(CLIENT_ID, redirect_uri=REDIRECT_URI)
    authorization_url, state = oauth.authorization_url(AUTH_URL)
    
    print(f"Visit this URL to authorize: {authorization_url}")
    authorization_response = input("Enter the callback URL: ")
    
    token = oauth.fetch_token(
        TOKEN_URL,
        client_secret=CLIENT_SECRET,
        authorization_response=authorization_response
    )
    return token
```

### Fetching Emails via API

```python
def fetch_emails(access_token, label_id='0', limit=100, offset=0):
    """Fetch emails from a specific label"""
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }
    
    params = {
        'LabelID': label_id,
        'Limit': limit,
        'Offset': offset
    }
    
    response = requests.get(
        'https://api.protonmail.ch/api/messages',
        headers=headers,
        params=params
    )
    
    return response.json()

def export_emails_to_json(access_token, output_file='emails.json'):
    """Export all emails to JSON for backup"""
    all_emails = []
    offset = 0
    limit = 100
    
    while True:
        emails = fetch_emails(access_token, limit=limit, offset=offset)
        if not emails.get('Messages'):
            break
            
        all_emails.extend(emails['Messages'])
        offset += limit
        print(f"Fetched {len(all_emails)} emails...")
    
    with open(output_file, 'w') as f:
        json.dump(all_emails, f, indent=2)
    
    print(f"Exported {len(all_emails)} emails to {output_file}")
    return all_emails
```

## Importing Emails into Proton Mail

For importing emails, the MBOX format works best. Proton Mail's import tool accepts MBOX files directly through the web interface. However, for automated or bulk imports, IMAP remains the most practical approach.

### Importing via IMAP

```python
def import_mbox_to_proton(mail, mbox_file, target_folder='Import'):
    """Import MBOX file to Proton Mail via IMAP"""
    # Create target folder if it doesn't exist
    try:
        mail.create(target_folder)
    except:
        pass
    
    mail.select(target_folder)
    
    with open(mbox_file, 'rb') as f:
        # Parse MBOX and add each message
        for message in email.message_from_file(f).walk():
            if message.get_payload():
                # Add to IMAP
                mail.append(target_folder, None, None, message.as_bytes())
    
    print(f"Imported messages from {mbox_file}")
```

## Automating Backups with Cron

For regular automated backups, combine these scripts with cron jobs:

```bash
# Add to crontab (crontab -e)
# Backup inbox daily at 2 AM
0 2 * * * cd /path/to/backup/scripts && python3 proton_backup.py >> /var/log/proton_backup.log 2>&1

# Cleanup old backups (keep last 7 days)
0 3 * * * find /path/to/backups -mtime +7 -delete
```

## Security Considerations

When handling Proton Mail exports, treat the data as sensitive. MBOX files contain plaintext email content, and API tokens provide full account access. Store exports encrypted using GPG or a dedicated encryption tool:

```bash
# Encrypt backup before storage
gpg --symmetric --cipher-algo AES256 inbox_backup.mbox

# Decrypt when needed
gpg --decrypt inbox_backup.mbox.gpg > inbox_backup.mbox
```



## Related Articles

- [Migrating From NordPass to Bitwarden](/privacy-tools-guide/migrating-from-nordpass-to-bitwarden-export-import-process-guide/)
- [Migrating from RoboForm to Bitwarden](/privacy-tools-guide/migrating-from-roboform-to-bitwarden-export-import-complete-/)
- [Bitwarden Vault Export Backup Guide](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [How to Export Passwords from Any Manager](/privacy-tools-guide/how-to-export-passwords-from-any-manager/)
- [How To Use Pgp Encrypted Email With Protonmail To Non Proton](/privacy-tools-guide/how-to-use-pgp-encrypted-email-with-protonmail-to-non-proton/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
