---
layout: default
title: "How To Set Up Proton Mail Bridge With Local Email Client"
description: "Install Proton Mail Bridge, log in with your Proton Mail credentials, then add the Bridge's local IMAP/SMTP server to your email client (Thunderbird, Apple"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-set-up-proton-mail-bridge-with-local-email-client-for/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---
---
layout: default
title: "How To Set Up Proton Mail Bridge With Local Email Client"
description: "Install Proton Mail Bridge, log in with your Proton Mail credentials, then add the Bridge's local IMAP/SMTP server to your email client (Thunderbird, Apple"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-set-up-proton-mail-bridge-with-local-email-client-for/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

Install Proton Mail Bridge, log in with your Proton Mail credentials, then add the Bridge's local IMAP/SMTP server to your email client (Thunderbird, Apple Mail, Neomutt): Bridge runs locally on your machine and automatically encrypts/decrypts messages while your email client communicates with it using standard protocols. This gives you full end-to-end encryption with the power of desktop email clients without sacrificing Proton Mail's zero-access encryption model.

## Key Takeaways

- **Configure the incoming mail server**: - Host Name: `127.0.0.1`
 - Port: `1143`
 - Use SSL: No (Bridge handles encryption internally)
5.
- **Configure the outgoing mail server**: - Host Name: `127.0.0.1`
 - Port: `1025`
 - Use SSL: No

Click Sign In and Apple Mail will connect through Bridge to Proton Mail.
- **This architecture lets you**: use powerful email clients like Neomutt, Apple Mail, or Thunderbird while retaining Proton Mail's zero-access encryption model.
- **Use full-disk encryption and**: keep your operating system updated.
- **Generate your IMAP/SMTP credentials**: then configure your preferred email client using the settings provided.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.

## What is Proton Mail Bridge?

Proton Mail Bridge runs locally on your machine and acts as an intermediary between your email client and Proton's servers. It handles the encryption and decryption automatically, meaning your email client communicates with Bridge using standard IMAP and SMTP protocols while Bridge manages the complex encryption layer.

The Bridge application stores your Proton Mail credentials locally and creates virtual IMAP and SMTP server configurations. Your email client never directly connects to Proton's cloud—instead, it talks to Bridge, which then communicates with Proton Mail using encrypted channels.

This architecture lets you use powerful email clients like Neomutt, Apple Mail, or Thunderbird while retaining Proton Mail's zero-access encryption model.

## Prerequisites

Before starting, ensure you have:

- A Proton Mail account (Plus or higher required for Bridge access)
- The Proton Mail Bridge application installed
- Your primary email address and password ready

Download Bridge from the Proton website or install via your system's package manager. The application is available for macOS, Windows, and Linux.

### Step 1: Install Proton Mail Bridge

On macOS with Homebrew:

```bash
brew install --cask proton-mail-bridge
```

On Linux, you can download the AppImage or use the Debian package:

```bash
wget https://protonmail.com/download/Bridge/ProtonMailBridge-3.0.0.deb
sudo dpkg -i ProtonMailBridge-3.0.0.deb
```

After installation, launch the Bridge application. You'll be prompted to log in with your Proton Mail credentials.

### Step 2: Configure Bridge for First Use

Open Proton Mail Bridge and sign in with your Proton Mail email and password. The application will generate IMAP and SMTP credentials specifically for your email client.

The Bridge interface displays connection details in this format:

```
IMAP Host: 127.0.0.1
IMAP Port: 1143
SMTP Host: 127.0.0.1
SMTP Port: 1025
Username: your-email@protonmail.com
Password: [Bridge-generated password]
```

These credentials are separate from your Proton Mail password. The Bridge application manages them, and you can regenerate them at any time through the Bridge interface if needed.

### Step 3: Set Up Apple Mail

For macOS users preferring Apple Mail over the Proton Mail web interface:

1. Open Apple Mail and go to **Mail > Add Account**
2. Select **Other Mail Account**
3. Enter your name, email address, and the Bridge password
4. Configure the incoming mail server:
 - Host Name: `127.0.0.1`
 - Port: `1143`
 - Use SSL: No (Bridge handles encryption internally)
5. Configure the outgoing mail server:
 - Host Name: `127.0.0.1`
 - Port: `1025`
 - Use SSL: No

Click **Sign In** and Apple Mail will connect through Bridge to Proton Mail.

### Step 4: Set Up Mozilla Thunderbird

Thunderbird provides excellent customization options for power users:

1. Go to **Edit > Account Settings > Account Actions > Add Mail Account**
2. Enter your name, email, and Bridge password
3. For incoming server configuration:
 - Incoming Server: `127.0.0.1`
 - Port: `1143`
 - SSL: None
 - Authentication: Normal password
4. For outgoing server:
 - Outgoing Server: `127.0.0.1`
 - Port: `1025`
 - SSL: None

Thunderbird will attempt automatic configuration, but manually entering these values ensures proper connection through Bridge.

### Step 5: Set Up Neomutt for Terminal Users

For developers who prefer terminal-based workflows, Neomutt works smoothly with Proton Mail Bridge:

Create or edit your Neomutt configuration file (`~/.muttrc`):

```muttrc
set imap_user = "your-email@protonmail.com"
set imap_pass = "[bridge-generated-password]"
set smtp_user = "your-email@protonmail.com"
set smtp_pass = "[bridge-generated-password]"
set smtp_url = "smtp://127.0.0.1:1025"
set imap_url = "imap://127.0.0.1:1143"
set folder = "imap://127.0.0.1:1143"
set spoolfile = "=INBOX"
set record = "=Sent"
set postponed = "=Drafts"
set trash = "=Trash"
```

Start Neomutt and it will connect through Bridge. You can also use `offlineimap` or `isync` to synchronize mail locally:

```bash
# Install isync
brew install isync

# Configure ~/.mbsyncrc
IMAPAccount proton
Host 127.0.0.1
Port 1143
User your-email@protonmail.com
Pass [bridge-password]

IMAPStore proton-remote
Account proton

LocalStore proton-local
Path ~/Mail/Proton/

MaildirStore proton-mail
Path ~/Mail/Proton/
Inbox ~/Mail/Proton/INBOX

Channel proton
Master :proton-remote:
Slave :proton-mail:
```

Run `mbsync proton` to synchronize your mail locally, then configure Neomutt to read from the local Maildir.

### Step 6: Enable Two-Factor Authentication for Bridge

For additional security, Bridge supports two-factor authentication through TOTP:

1. In the Bridge application, go to **Settings > Security**
2. Enable **Two-Factor Authentication**
3. Scan the QR code with your authenticator app
4. Enter the verification code

Your email client will now require both the Bridge password and a current TOTP code to connect.

### Step 7: Manage Multiple Accounts

Bridge supports multiple Proton Mail accounts simultaneously. Click the **+** button in the Bridge interface to add additional accounts. Each account receives its own IMAP/SMTP credentials.

To use multiple accounts in your email client, create separate configurations for each identity:

```muttrc
# Primary account
set imap_user = "primary@protonmail.com"
set smtp_user = "primary@protonmail.com"

# Define alternate accounts with macros
macro index,pager \cb "<change-folder>proton-secondary/INBOX<enter>" "Switch to secondary account"
```

## Troubleshooting Connection Issues

If your email client fails to connect through Bridge, verify these common issues:

**Port conflicts**: Ensure no other service is using ports 1143 (IMAP) or 1025 (SMTP). Check with:

```bash
lsof -i :1143
lsof -i :1025
```

**Bridge not running**: Bridge must remain open for your email client to connect. Consider launching Bridge at system startup.

**Credential expiration**: Bridge credentials can expire or become invalid. Open Bridge and check the account status—regenerate credentials if needed.

**Firewall settings**: Some system firewalls may block local connections to Bridge. Verify that your firewall permits connections to 127.0.0.1 on the specified ports.

## Security Considerations

Using Proton Mail Bridge maintains Proton's end-to-end encryption guarantees because:

- Your password never leaves the Bridge application
- All communication between Bridge and Proton's servers is encrypted
- Your email client sees only decrypted content locally
- Bridge operates entirely on your machine, not in the cloud

However, ensure your local machine is secure since decrypted emails exist in your email client's cache. Use full-disk encryption and keep your operating system updated.

## Performance Tips

For users with large mailboxes, consider these optimizations:

**Enable caching in your email client**: Thunderbird and Apple Mail can store local copies of messages, reducing Bridge overhead.

**Use offlineimap or isync**: Synchronize mail periodically rather than maintaining a live connection:

```bash
# Add to crontab for periodic sync
*/15 * * * * /usr/local/bin/mbsync proton
```

**Limit synchronized folders**: Configure your client to sync only essential folders, excluding large folders like Sent or Archive if unnecessary.

## Getting Started

Begin by downloading Proton Mail Bridge and logging in with your account. Generate your IMAP/SMTP credentials, then configure your preferred email client using the settings provided. Test sending and receiving messages to verify the connection works correctly.

Once configured, you gain the flexibility of using any email client while maintaining Proton Mail's encryption standards. The Bridge application runs quietly in the background, handling all encryption transparently.

## Frequently Asked Questions

**How long does it take to set up proton mail bridge with local email client?**

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

- [Proton Mail Account Inheritance How Encrypted Email Provider](/privacy-tools-guide/proton-mail-account-inheritance-how-encrypted-email-provider/)
- [Set Up Mail In A Box Private Email Server Complete 2026](/privacy-tools-guide/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Set Up Own Email Server For Maximum Privacy Using Mail In](/privacy-tools-guide/how-to-set-up-own-email-server-for-maximum-privacy-using-mail-in-box/)
- [Proton Drive Bridge Desktop Integration Guide](/privacy-tools-guide/proton-drive-bridge-desktop-integration-guide/)
- [Proton Drive Linux Client Setup Guide 2026](/privacy-tools-guide/proton-drive-linux-client-setup-guide-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
