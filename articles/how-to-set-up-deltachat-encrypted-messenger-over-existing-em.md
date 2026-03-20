---
layout: default
title: "How To Set Up Deltachat Encrypted Messenger Over Existing Em"
description: "A practical guide for developers and power users to configure DeltaChat for end-to-end encrypted messaging using your existing email infrastructure."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-deltachat-encrypted-messenger-over-existing-em/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

Set up DeltaChat by installing the application, adding your existing email account, and enabling Autocrypt encryption to create end-to-end encrypted messaging over email infrastructure. DeltaChat works with any IMAP-compatible email provider without server setup, making it ideal for developers who want encryption without infrastructure maintenance. The email-based architecture means you retain access through any email client and avoid vendor lock-in, though metadata (subject lines, sender/recipient addresses) remains visible to email providers.

## Understanding DeltaChat's Architecture

DeltaChat operates on a decentralized model by using email protocols with automatic encryption. Rather than maintaining its own network, DeltaChat uses your existing email provider as the transport layer while implementing Autocrypt to handle key management and message encryption. This approach means you retain full ownership of your communication without depending on a single service provider.

When you send a message through DeltaChat, it travels as an encrypted email to your recipient's email address. The recipient's DeltaChat client automatically detects and decrypts incoming messages. This architecture provides several advantages: your messages are accessible from any email client, you can communicate with non-DeltaChat users via regular email, and there's no vendor lock-in.

## Prerequisites

Before configuring DeltaChat, ensure you have access to an email account that supports IMAP and SMTP. Most providers—including Gmail, ProtonMail, self-hosted solutions like Mailu or DockerMail, and standard email services—work with DeltaChat.

For developers seeking maximum privacy, consider running a personal email server. A minimal Postfix and Dovecot setup on a VPS provides complete control over your communication infrastructure.

## Configuring DeltaChat with Your Email Provider

### Desktop Client Setup

Download the DeltaChat desktop client from the official repository or install via your system's package manager. On Linux, you can use:

```bash
# Debian/Ubuntu
sudo apt install deltachat

# Fedora
sudo dnf install deltachat

# Arch Linux
sudo pacman -S deltachat
```

Launch the application and navigate to the account configuration. Select "Connect Email Account" and choose your provider from the list, or manually configure using these settings:

```
IMAP Server: yourmail.example.com
IMAP Port: 993
IMAP Security: SSL/TLS
SMTP Server: yourmail.example.com  
SMTP Port: 587
SMTP Security: STARTTLS
```

For Gmail users, you'll need to generate an App Password since DeltaChat cannot use your primary Google credentials. Navigate to your Google Account security settings, enable 2-Step Verification, and create an App Password specifically for DeltaChat.

### Android and iOS Configuration

Mobile users can install DeltaChat from F-Droid (recommended for privacy-conscious users) or the Google Play Store. The setup process mirrors the desktop experience—enter your email credentials and let DeltaChat handle the connection details automatically.

For advanced users configuring custom servers, tap the three-dot menu and select "Manual Settings" to access advanced configuration options.

## Verifying Encryption Setup

After initial configuration, DeltaChat generates encryption keys automatically. To verify your setup is functioning correctly:

1. Start a chat with a contact who also uses DeltaChat
2. Look for the green lock icon in the conversation header
3. Tap the lock icon to view key fingerprints

DeltaChat implements the Autocrypt Level 1 specification, which automatically negotiates encryption between clients without requiring manual key exchange. Each device generates its own key pair, and keys are shared automatically when you first message another DeltaChat user.

### Checking Key Fingerprints

For paranoid security verification, compare key fingerprints through an out-of-band channel:

```
# View your fingerprint in DeltaChat desktop
Settings → Encryption → Show My Fingerprint
```

Both parties should verify they see matching fingerprints to confirm there's no man-in-the-middle attack.

## Advanced Configuration Options

### Custom Server Configuration

Developers hosting their own mail servers can fine-tune DeltaChat's behavior through advanced settings. Access these via:

```
Settings → Advanced → Custom Settings
```

Key settings include:

- `e2ee_enabled` — Enable or disable end-to-end encryption globally
- `autokey` — Automatic key generation for new contacts
- `bcc_self` — Receive copies of messages you send

### Self-Hosted Email Server Recommendations

For users seeking complete infrastructure control, the following self-hosted solutions integrate well with DeltaChat:

- **Mailu** — Docker-based mail server with webmail, IMAP, and SMTP
- **Mailcow** — Feature-rich mail server with a modern web interface
- **DockerMail** — Lightweight containerized mail solution
- **Postfix + Dovecot** — Manual setup offering maximum customization

A minimal Postfix configuration for DeltaChat compatibility requires:

```bash
# /etc/postfix/main.cf
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtpd_use_tls = yes
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
```

## Troubleshooting Common Issues

### Connection Failures

If DeltaChat fails to connect, verify your credentials and server settings. Common issues include:

- **Port blocking** — Ensure your firewall allows IMAP (993) and SMTP (587/465) traffic
- **SSL certificate errors** — Check that your server's TLS certificate is valid and not self-signed for production use
- **Authentication failures** — Confirm your credentials work in a standard email client

### Encryption Not Working

When the green lock icon doesn't appear:

1. Verify your contact is also using DeltaChat
2. Check that both parties have encryption enabled in settings
3. Try sending a new message—Autocrypt activates on first contact

### Message Delivery Delays

DeltaChat checks for new messages periodically. You can reduce latency by adjusting the sync interval in settings, though more frequent checks increase server load and battery consumption on mobile devices.

## Autocrypt Key Management Deep Dive

DeltaChat implements the Autocrypt standard, which automatically manages encryption keys without user intervention:

```python
# Understanding Autocrypt key exchange
autocrypt_process = {
    "first_message": {
        "sender": "Generates keypair locally",
        "includes": "Public key in message headers",
        "recipient": "Receives and validates public key"
    },
    "second_message": {
        "recipient": "Replies with their public key",
        "result": "Bidirectional encryption now active"
    },
    "ongoing": {
        "verification": "Optional fingerprint comparison",
        "mechanism": "Out-of-band verification (phone, in-person)"
    }
}
```

For paranoid users, verify fingerprints through an independent channel:

```bash
# Export your public key for out-of-band verification
# In DeltaChat: Settings → Encryption → Export Public Key
# Share fingerprint via voice call or in-person meeting
```

## Integration with Self-Hosted Email Infrastructure

For maximum control, run DeltaChat against self-hosted mail servers:

```bash
# Minimal Postfix + Dovecot setup for DeltaChat
# Install on a VPS (Ubuntu 22.04 example)

# 1. Install prerequisites
sudo apt update
sudo apt install postfix dovecot-core dovecot-imapd certbot python3-certbot-nginx

# 2. Configure Postfix (simplified /etc/postfix/main.cf)
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtpd_use_tls = yes

# 3. Generate SSL certificate
sudo certbot certonly --standalone -d mail.example.com

# 4. Configure Dovecot (/etc/dovecot/conf.d/10-ssl.conf)
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem

# 5. Create user accounts
sudo useradd -m -s /usr/sbin/nologin user@example.com
echo "user@example.com:$(openssl passwd -crypt password)" | sudo tee -a /etc/dovecot/users

# 6. Test DeltaChat connection
# Connect to mail.example.com:993 (IMAP) and mail.example.com:587 (SMTP)
```

## Multi-Device DeltaChat Setup

Managing DeltaChat across multiple devices requires careful key synchronization:

```python
# Device synchronization strategy
device_setup = {
    "primary_device": {
        "generates": "Master keypair",
        "stores": "Backup code (write down and secure)",
        "shares": "Public key to all secondary devices"
    },
    "secondary_device": {
        "connects": "Same email account",
        "imports": "Public key from primary",
        "creates": "Device-specific keypair",
        "result": "Multiple valid keys for same account"
    },
    "important": "Each device has different keys; primary device key is irreplaceable"
}
```

Export your backup code immediately after setup:

```bash
# In DeltaChat: Settings → Encryption → Backup
# Write down the 11-word backup code
# Store in secure location separate from devices
# Can be used to restore if device is lost
```

## Comparing DeltaChat to Alternatives

When evaluating encrypted messaging solutions:

| Feature | DeltaChat | Signal | ProtonMail | Wickr |
|---------|-----------|--------|------------|-------|
| Uses email infrastructure | Yes | No | Yes | No |
| Vendor lock-in risk | Low | Medium | Medium | High |
| Zero-knowledge | Yes | Yes | Yes | Yes |
| Metadata protection | Partial | High | Partial | High |
| Requires phone number | No | Yes | No | No |
| Self-hosted option | Partial | No | No | No |
| Cost | Free | Free | Free-paid | Paid |

DeltaChat stands out for its use of existing email infrastructure—you can access conversations from any email client, even if DeltaChat is unavailable.

## Practical DeltaChat Workflows

### For Journalists

```bash
# Journalist-to-source encrypted communication
# 1. Create dedicated DeltaChat profile
# 2. Share email address publicly on website/social media
# 3. Sources contact via email → automatically encrypted
# 4. No phone number exposure, no centralized server
```

### For Organizations

```bash
# Team communication setup
# 1. Assign DeltaChat email to each team member
# 2. Create group chats in DeltaChat
# 3. All messages encrypted end-to-end
# 4. Requires no infrastructure beyond existing mail server
```

### For Privacy-Conscious Developers

```python
# Integration example: Send encrypted DeltaChat message
import subprocess
import os

def send_deltachat_message(recipient_email: str, message: str):
    """Send encrypted message via DeltaChat"""

    # Requires DeltaChat CLI (development version)
    result = subprocess.run([
        'deltachat',
        'send',
        recipient_email,
        message
    ], capture_output=True)

    if result.returncode == 0:
        return {"status": "sent", "encrypted": True}
    else:
        return {"status": "failed", "error": result.stderr.decode()}

# Usage
send_deltachat_message(
    "recipient@example.com",
    "This message is automatically encrypted"
)
```

## Security Considerations

While DeltaChat provides strong end-to-end encryption, several supplementary practices enhance your security posture:

- Enable two-factor authentication on your email provider
- Use a password manager for credential storage
- Regularly backup your DeltaChat database and backup code
- Consider using a dedicated email account for sensitive communications
- Keep DeltaChat and your email client updated
- Monitor device access to your email account
- Review email forwarding rules and authorized apps regularly

Metadata—including subject lines, sender, and recipient addresses—remains visible to email providers. For complete anonymity, combine DeltaChat with a privacy-focused email provider (ProtonMail, Tutanota) and consider using Tor or a VPN for email access.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
