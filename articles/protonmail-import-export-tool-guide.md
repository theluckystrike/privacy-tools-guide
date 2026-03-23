---
layout: default
title: "ProtonMail Import Export Tool Guide"
description: "Export Proton Mail data using three methods: the web interface (Settings > Download my data) for one-time MBOX exports, the IMAP Bridge (brew install --cask"
date: 2026-03-15
last_modified_at: 2026-03-22
author: "Privacy Tools Guide"
permalink: /protonmail-import-export-tool-guide/
categories: [guides]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Export Proton Mail data using three methods: the web interface (Settings > Download my data) for one-time MBOX exports, the IMAP Bridge (`brew install --cask proton-mail-bridge`) for continuous desktop client sync, or the Proton Mail API for automated developer integrations. For imports, upload MBOX files through the web interface or push messages via IMAP. This guide covers step-by-step configuration for each method, including Python automation scripts and cron-based backup scheduling.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand ProtonMail's Data Export Options

Proton Mail provides several pathways for exporting data, each with different capabilities and use cases. The primary methods include the web interface export, IMAP Bridge for continuous sync, and API-based extraction for automated workflows.

The web interface allows you to export individual folders or your entire mailbox as MBOX format. This works well for one-time migrations but becomes tedious if you need regular automated backups. For developers building integration tools, the Proton Mail API provides programmatic access to emails, contacts, and calendar events.

A fourth option worth mentioning is the Proton Mail Import-Export app, a standalone desktop application available for Windows, macOS, and Linux. Unlike the web interface, the Import-Export app can handle large mailboxes more reliably by processing messages in batches and resuming interrupted exports. Download it from proton.me/mail/bridge.

### Step 2: Exporting via Proton Mail Web Interface

The most straightforward export method uses the Proton Mail web application. Navigate to Settings → Download my data → Export. You can select specific folders or export everything including emails, contacts, and calendar entries.

The export generates a MBOX file that works with most email clients. For large mailboxes, expect the process to take several hours since Proton processes exports asynchronously for security reasons. Once complete, Proton sends an email notification with a secure download link. This link expires after 4 days, so download the file promptly.

**Supported export formats:**
- Emails: MBOX (compatible with Thunderbird, Apple Mail, mutt)
- Contacts: VCF (vCard format, compatible with most address books)
- Calendar: ICS (iCalendar format, compatible with Google Calendar, Apple Calendar)

### Step 3: Use Proton Mail IMAP Bridge

For continuous synchronization with desktop email clients, Proton Mail offers the IMAP Bridge application. This tool enables you to connect Thunderbird, Apple Mail, or any IMAP-compatible client to your Proton Mail account. The bridge runs locally and decrypts messages on-device before passing them to your email client, maintaining end-to-end encryption integrity.

### Installing and Configuring IMAP Bridge

```bash
# macOS installation via Homebrew
brew install --cask proton-mail-bridge

# Debian/Ubuntu
sudo apt install protonmail-bridge

# Fedora/RHEL
sudo rpm -i protonmail-bridge-*.rpm
```

After installation, launch the bridge and authenticate with your Proton Mail credentials. The bridge provides separate IMAP and SMTP credentials specifically for external client connections. These are bridge-specific app passwords — not your main Proton Mail password. Configure your email client using these credentials:

```
IMAP Server: 127.0.0.1
IMAP Port: 1143
SMTP Server: 127.0.0.1
SMTP Port: 1025
Security: None (bridge handles encryption locally)
```

**Configuring Thunderbird with IMAP Bridge:**

1. Open Thunderbird → Account Settings → Account Actions → Add Mail Account
2. Enter your Proton Mail address and the bridge app password (not your main password)
3. Manually set the server settings to 127.0.0.1 with ports 1143 (IMAP) and 1025 (SMTP)
4. Set security to None — bridge encrypts internally

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

### Listing Available Folders

Before exporting, list all folders to get exact folder names (Proton Mail uses internal names that differ from display names):

```python
def list_folders(mail):
    """List all available IMAP folders"""
    typ, folder_list = mail.list()
    for folder in folder_list:
        print(folder.decode())

mail = connect_imap_bridge()
list_folders(mail)
# Output example:
# (\HasNoChildren) "/" "INBOX"
# (\HasNoChildren) "/" "Sent"
# (\HasNoChildren) "/" "Drafts"
# (\HasNoChildren) "/" "Spam"
```

### Step 4: API-Based Export for Developers

Proton Mail's API offers the most flexibility for custom integrations. To use the API, you need to create an app in your Proton Mail account settings and obtain OAuth credentials. Note that Proton's API uses a custom authentication flow — standard OAuth libraries need adaptation.

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

### Step 5: Importing Emails into Proton Mail

For importing emails, the MBOX format works best. Proton Mail's import tool accepts MBOX files directly through the web interface under Settings → Import via IMAP/MBOX. However, for automated or bulk imports, IMAP remains the most practical approach.

The standalone Import-Export app handles large migrations more reliably than the web interface. It supports importing from Gmail, Outlook, Yahoo Mail, and any IMAP server, with folder mapping and duplicate detection built in.

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

### Step 6: Automate Backups with Cron

For regular automated backups, combine these scripts with cron jobs:

```bash
# Add to crontab (crontab -e)
# Backup inbox daily at 2 AM
0 2 * * * cd /path/to/backup/scripts && python3 proton_backup.py >> /var/log/proton_backup.log 2>&1

# Cleanup old backups (keep last 7 days)
0 3 * * * find /path/to/backups -mtime +7 -delete
```

For macOS, use launchd instead of cron for more reliable scheduling. Create a plist at `~/Library/LaunchAgents/com.protonbackup.plist` with an interval key set to 86400 (seconds) for daily execution.

## Security Considerations

When handling Proton Mail exports, treat the data as sensitive. MBOX files contain plaintext email content, and API tokens provide full account access. Store exports encrypted using GPG or a dedicated encryption tool:

```bash
# Encrypt backup before storage
gpg --symmetric --cipher-algo AES256 inbox_backup.mbox

# Decrypt when needed
gpg --decrypt inbox_backup.mbox.gpg > inbox_backup.mbox
```

**Additional security recommendations:**

- Store bridge app passwords in a password manager like Bitwarden or 1Password, not in plaintext config files
- Use environment variables for API credentials in scripts rather than hardcoding them
- Rotate bridge app passwords quarterly — revoke old ones through Proton Mail account settings
- For team environments, use dedicated Proton Mail accounts for automated export jobs rather than personal accounts
- Consider storing MBOX backups on an encrypted volume (VeraCrypt or LUKS) rather than unencrypted cloud storage

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to complete this setup?**

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

- [How to Export Passwords from Any Manager](/how-to-export-passwords-from-any-manager/)
- [Bitwarden Vault Export Backup Guide](/bitwarden-vault-export-backup-guide/)
- [How To Set Up Proton Mail Bridge With Local Email Client](/how-to-set-up-proton-mail-bridge-with-local-email-client-for/)
- [Using curl for LinkedIn API](/social-media-data-request-download-guide-2026/)
- [ProtonMail vs Skiff Mail Comparison: A Developer Guide](/protonmail-vs-skiff-mail-comparison/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
