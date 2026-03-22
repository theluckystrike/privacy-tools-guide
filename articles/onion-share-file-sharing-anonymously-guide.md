---
layout: default
title: "Onion Share File Sharing Anonymously Guide"
description: "Onion Share is an open-source tool that creates temporary .onion services allowing you to share files anonymously through Tor without exposing your IP address"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /onion-share-file-sharing-anonymously-guide/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Onion Share is an open-source tool that creates temporary .onion services allowing you to share files anonymously through Tor without exposing your IP address, recipient IP, or any metadata about the transfer. Install via Homebrew on macOS or your package manager on Linux, then use the simple GUI or CLI to select files, start sharing, and distribute the unique .onion URL to recipients. This guide covers installation, command-line workflows, persistent sharing, and practical integration patterns for developers and journalists.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install Onion Share

Onion Share is available on macOS, Linux, and Windows. On macOS, install via Homebrew:

```bash
brew install onionshare
```

On Linux, install from your distribution's package manager or download the AppImage:

```bash
wget https://github.com/micahflee/onionshare/releases/download/v2.6.1/OnionShare-2.6.1.AppImage
chmod +x OnionShare-2.6.1.AppImage
./OnionShare-2.6.1.AppImage
```

For headless servers or CLI automation, install the Python package:

```bash
pip install onionshare-cli
```

Verify the installation:

```bash
onionshare-cli --version
```

### Step 2: Understand How Onion Share Works

Onion Share creates a Tor hidden service pointed at files or folders on your local machine. When you start a sharing session, Onion Share generates a unique .onion URL that you share with recipients. The connection travels entirely through the Tor network, providing end-to-end encryption and anonymity for both sender and receiver.

The key advantage over cloud-based file sharing is that no intermediary server stores your files. The data transfers directly from your computer to the recipient's computer through Tor's encrypted circuits.

### Step 3: Basic File Sharing

Share a single file using the graphical interface or CLI. With onionshare-cli:

```bash
onionshare-cli --verbose /path/to/document.pdf
```

The output displays the .onion URL and a privacy flag:

```
OnionShare service starting at:
http://onionshare:password@abc1234567890.onion/file.pdf
```

Share multiple files by specifying a directory:

```bash
onionshare-cli --verbose ~/SharedFiles/
```

Recipients visit the .onion URL in Tor Browser. They download files directly without installing Onion Share themselves.

### Step 4: Persistent Shared Folders

For recurring file sharing without regenerating URLs, configure a persistent shared folder. Create a configuration file:

```yaml
# persistent-config.yaml
persistent_folder: /home/user/always-available
allow_downloads: true
allow_upload: false
shutdown_timeout: 60
```

Start Onion Share with the persistent flag:

```bash
onionshare-cli --persistent persistent-config.yaml
```

This creates a stable .onion address that remains active until you stop the service. Files in the designated folder stay available for download across multiple sessions.

### Step 5: Command-Line Options and Automation

Onionshare-cli provides numerous options for scripted workflows:

```bash
# Share with automatic shutdown after last download
onionshare-cli --auto-shutdown --verbose document.zip

# Require password authentication
onionshare-cli --password "your-secure-password" sensitive-data.tar.gz

# Limit to single download
onionshare-cli --stay-open --single-download large-file.iso

# Custom port (useful for containerized environments)
onionshare-cli --port 12345 --verbose files/
```

Combine options for specific use cases:

```bash
#!/bin/bash
# automated-share.sh

PASSWORD="${1:-}"
FILE="$2"

if [ -z "$FILE" ]; then
    echo "Usage: $0 <password> <file>"
    exit 1
fi

if [ -n "$PASSWORD" ]; then
    onionshare-cli --password "$PASSWORD" --auto-shutdown --verbose "$FILE"
else
    onionshare-cli --auto-shutdown --verbose "$FILE"
fi
```

Make it executable and use it:

```bash
chmod +x automated-share.sh
./automated-share.sh "securepass123" ~/documents/report.pdf
```

### Step 6: Integrate with Tor Daemon

For advanced deployments, run Onion Share connected to an existing Tor daemon instead of spawning its own Tor process. This reduces resource usage and provides centralized control:

```bash
# Configure Onion Share to use system Tor
onionshare-cli --tor-control-port 9051 --verbose document.pdf
```

Configure your torrc to enable the control port:

```
ControlPort 9051
CookieAuthentication 1
```

This approach is useful for servers that already run Tor for other services.

### Step 7: Web Server Mode

Onion Share can serve a simple web interface with directory listings:

```bash
onionshare-cli --website --verbose ~/public-folder/
```

This mode serves all files in the specified directory with automatic index generation. Recipients browse and select files through a web interface in Tor Browser.

## Security Considerations

Onion Share provides strong anonymity guarantees, but follow these practices for optimal security:

Always verify the .onion URL through a separate channel when sharing sensitive files. Recipients should confirm they received the correct URL to prevent URL substitution attacks.

Use password protection for sensitive transfers:

```bash
onionshare-cli --password "unique-per-session-password" confidential.7z
```

Generate unique passwords for each sharing session rather than reusing passwords.

Disable uploads unless necessary. The default configuration denies uploads—only change this when you explicitly need to receive files.

Be aware that your IP address remains hidden, but timing metadata (file size, transfer duration) may reveal information about the transferred content. For extremely sensitive transfers, consider padding files or using encryption before sharing.

## Troubleshooting

If connections fail, verify your Tor configuration:

```bash
tor --verify-config
```

Check that Tor is running and the control port is accessible:

```bash
nc -z localhost 9051 && echo "Tor control port open"
```

Onion Share may fail if your Tor instance lacks sufficient directory authorities. Add bridges for restricted networks:

```bash
onionshare-cli --use-bridges --verbose file.txt
```

For persistent issues, check Onion Share's debug logs:

```bash
onionshare-cli --debug --verbose file.txt
```

### Step 8: Use Cases for Developers

Onion Share serves various developer workflows:

Transfer sensitive logs or debug files from production environments without exposing server IPs. Share via a temporary .onion URL that auto-shuts down after download.

Distribute pre-release software to trusted testers while maintaining anonymity for both parties. Testers download without revealing their IP addresses.

Exchange cryptographic keys or credentials with colleagues across organizational boundaries. The Tor network provides encryption and anonymity without requiring accounts on third-party services.

Automated backup transfer to isolated machines:

```bash
#!/bin/bash
# backup-to-isolated.sh

BACKUP_FILE="/backup/$(date +%Y%m%d).tar.gz"
ONION_URL=$(onionshare-cli --auto-shutdown --verbose "$BACKUP_FILE" | grep "http.*onion" | head -1)

# Send URL via secure channel (Signal, encrypted email, etc.)
echo "Backup available at: $ONION_URL"
```

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

- [Onionshare Secure File Sharing Over Tor Network Setup](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Secure File Sharing with OnionShare](/privacy-tools-guide/onionshare-secure-file-sharing-guide/)
- [Secure File Sharing Tools Comparison: E2E Encrypted](/privacy-tools-guide/secure-file-sharing-tools-comparison/)
- [Syncthing Setup Guide for Private File Sync](/privacy-tools-guide/syncthing-setup-guide-private-file-sync/)
- [How To Use Age Encryption For Secure File Sharing Command](/privacy-tools-guide/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
