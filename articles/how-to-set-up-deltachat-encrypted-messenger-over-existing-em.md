---
layout: default
title: "How To Set Up Deltachat Encrypted Messenger Over Existing Em"
description: "A practical guide for developers and power users to configure DeltaChat for end-to-end encrypted messaging using your existing email infrastructure."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-deltachat-encrypted-messenger-over-existing-em/
categories: [guides]
tags: [tools]
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

## Security Considerations

While DeltaChat provides strong end-to-end encryption, several supplementary practices enhance your security posture:

- Enable two-factor authentication on your email provider
- Use a password manager for credential storage
- Regularly backup your DeltaChat database and keys
- Consider using a dedicated email account for sensitive communications

Metadata—including subject lines, sender, and recipient addresses—remains visible to email providers. For complete anonymity, combine DeltaChat with a privacy-focused email provider and consider using Tor or a VPN.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
