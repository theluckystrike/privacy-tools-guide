---
layout: default
title: "Onion Share File Sharing Anonymously Guide"
description: "A practical guide to using Onion Share for anonymous, encrypted file sharing. Learn setup, configuration, persistent URLs, and advanced workflows for developers."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /onion-share-file-sharing-anonymously-guide/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
---

{% raw %}

Onion Share is an open-source tool that enables secure, anonymous file sharing through the Tor network. Unlike traditional file transfer methods that expose your IP address and metadata, Onion Share creates temporary .onion services that allow recipients to download files directly from your machine without either party revealing their network location. This guide covers installation, command-line usage, persistent sharing, and integration workflows for developers and power users.

## Installing Onion Share

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

## Understanding How Onion Share Works

Onion Share creates a Tor hidden service pointed at files or folders on your local machine. When you start a sharing session, Onion Share generates a unique .onion URL that you share with recipients. The connection travels entirely through the Tor network, providing end-to-end encryption and anonymity for both sender and receiver.

The key advantage over cloud-based file sharing is that no intermediary server stores your files. The data transfers directly from your computer to the recipient's computer through Tor's encrypted circuits.

## Basic File Sharing

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

## Persistent Shared Folders

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

## Command-Line Options and Automation

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

## Integrating with Tor Daemon

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

## Web Server Mode

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

## Use Cases for Developers

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

## Conclusion

Onion Share provides a privacy-first alternative to conventional file sharing. Its Tor-based architecture eliminates the need for third-party storage, giving you control over your data while protecting both sender and recipient identity. Start with basic file sharing, then explore persistent folders, CLI automation, and Tor daemon integration as your needs grow.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
