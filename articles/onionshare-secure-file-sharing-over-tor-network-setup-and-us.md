---
---
layout: default
title: "Onionshare Secure File Sharing Over Tor Network Setup"
description: "OnionShare uses Tor hidden services to enable direct, peer-to-peer file sharing that exposes neither your IP address nor any files to third-party servers"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /onionshare-secure-file-sharing-over-tor-network-setup-and-us/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

OnionShare uses Tor hidden services to enable direct, peer-to-peer file sharing that exposes neither your IP address nor any files to third-party servers, making it ideal for journalists, whistleblowers, and anyone sharing sensitive data. Install Tor and OnionShare via your package manager or Homebrew, then use the CLI to select files, start sharing, and securely distribute the temporary .onion URL to recipients. This guide covers complete setup, CLI automation, advanced usage patterns, and security best practices for developers and high-security environments.

## Prerequisites and Initial Setup

Before installing OnionShare, ensure you have a working Tor installation. Most Linux distributions include Tor in their repositories, but for the latest stable version, add the Tor Project's repository:

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install tor

# Verify Tor is running
tor --version
```

On macOS, install Tor via Homebrew:

```bash
brew install tor
brew services start tor
```

Windows users can download the Tor Browser bundle, which includes the Tor daemon needed for OnionShare to function.

### Step 1: Install OnionShare

OnionShare provides both GUI and CLI versions. For server environments and automation, install the CLI version:

```bash
# Install via pip (requires Python 3.7+)
pip install onionshare-cli

# Verify installation
onionshare-cli --version
```

For desktop environments, download the appropriate package from the official GitHub repository:

```bash
# Example: Download latest Linux release
wget $(curl -s https://api.github.com/repos/micahflee/onionshare/releases/latest | grep -o "https://.*OnionShare.*\.AppImage" | head -1)
chmod +x OnionShare-*.AppImage
```

### Step 2: Understand the Tor Network Connection

OnionShare operates by creating a temporary Tor hidden service that points to files or directories on your local machine. When you start a sharing session, the tool generates a unique `.onion` URL valid only for that transfer. The recipient connects through the Tor network, establishing an end-to-end encrypted tunnel directly to your machine.

This architecture provides several security advantages over traditional file sharing:

- No intermediary servers store your data
- Your IP address remains hidden from the recipient
- The transfer metadata is protected by Tor's circuit architecture
- Files transfer directly between endpoints

### Step 3: Basic File Sharing Workflow

Start a simple file share using the CLI:

```bash
# Share a single file
onionshare-cli --verbose /path/to/document.pdf

# Share multiple files
onionshare-cli --verbose /path/to/file1.txt /path/to/file2.zip

# Share an entire directory
onionshare-cli --verbose /path/to/folder/
```

The CLI outputs a unique URL similar to `http://abcd1234567890.onion/`. Share this URL with your recipient through a secure channel (Signal, encrypted email, or面对面传递). The connection remains active until either the recipient completes the download or you terminate the process.

### Step 4: Command-Line Options for Power Users

OnionShare CLI offers numerous options for controlling transfer behavior:

```bash
# Auto-shutdown after successful transfer
onionshare-cli --auto-shutdown /path/to/file

# Set a custom port
onionshare-cli --port 12345 /path/to/file

# Enable persistent URL (reusable after restart)
onionshare-cli --persistent /path/to/folder

# Require password protection
onionshare-cli --password "your-secure-password" /path/to/file

# Limit download count
onionshare-cli --download-limit 5 /path/to/file

# Receive files instead of sending
onionshare-cli --receive /path/to/upload/directory
```

The `--receive` mode is particularly useful for secure document collection. Recipients upload files directly to your machine through Tor, eliminating the need for third-party file upload services.

### Step 5: Automate OnionShare in Scripts

For regular file sharing workflows, integrate OnionShare into shell scripts:

```bash
#!/bin/bash
# secure-share.sh - Automated secure file sharing

FILE="$1"
NOTIFY_URL="$2"  # Webhook or notification service

# Generate share URL with auto-shutdown
SHARE_URL=$(onionshare-cli --auto-shutdown --verbose "$FILE" 2>&1 | \
    grep -oP 'http://[a-z2-7]{56}.onion/\S+' | head -1)

if [ -n "$SHARE_URL" ]; then
    # Send notification with share URL
    curl -X POST "$NOTIFY_URL" -d "{\"url\": \"$SHARE_URL\"}"
    echo "Share URL: $SHARE_URL"
else
    echo "Failed to create share"
    exit 1
fi
```

Schedule recurring shares using cron:

```bash
# Edit crontab
crontab -e

# Add scheduled share (every day at 2 AM)
0 2 * * * /usr/local/bin/secure-share.sh /backup/daily.tar.gz https://hooks.example.com/notify
```

## Security Considerations

While OnionShare provides strong anonymity guarantees, follow these best practices:

**Network Isolation**: Consider running OnionShare from an isolated network namespace or VPN to prevent IP leaks through non-Tor traffic:

```bash
# Create network namespace for OnionShare
sudo ip netns add onionshare
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns onionshare
```

**File Metadata Stripping**: Remove metadata from files before sharing:

```bash
# Remove EXIF and other metadata
exiftool -all= document.pdf
# Or use mat2 for complete metadata removal
pip install mat2
mat2 document.pdf
```

**Verification**: Generate checksums for shared files to ensure integrity:

```bash
# Generate SHA256 checksum
sha256sum /path/to/file > file.sha256

# Share both files via OnionShare
onionshare-cli /path/to/file /path/to/file.sha256
```

## Troubleshooting Common Issues

Connection failures typically stem from Tor configuration problems. Verify your Tor daemon is running and accessible:

```bash
# Check Tor status
systemctl status tor

# Test Tor connectivity
curl --socks5 localhost:9050 https://check.torproject.org/api/ip
```

If OnionShare fails to start, examine the verbose output:

```bash
onionshare-cli --verbose --debug /path/to/file
```

For persistent sharing issues, ensure the data directory has proper permissions:

```bash
mkdir -p ~/.local/share/onionshare
chmod 700 ~/.local/share/onionshare
```

## Advanced: Tor Daemon Integration

For high-availability setups, integrate with a persistent Tor daemon rather than relying on OnionShare's built-in Tor process:

```bash
# Configure Tor to allow control connections
# Add to /etc/tor/torrc:
ControlPort 9051
CookieAuthentication 1

# Restart Tor
sudo systemctl restart tor

# Run OnionShare using system Tor
onionshare-cli --tor-control-port 9051 --use-system-tor /path/to/file
```

This approach provides better resource management for servers handling multiple concurrent shares.

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

- [Best Secure File Sharing Tools for Teams Handling.](/privacy-tools-guide/best-secure-file-sharing-tools-for-teams-handling-sensitive-data/)
- [How to Set Up Secure File Sharing for Sensitive Documents](/privacy-tools-guide/how-to-set-up-secure-file-sharing-for-sensitive-documents/)
- [How To Use Age Encryption For Secure File Sharing Command Li](/privacy-tools-guide/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
- [Secure File Sharing Tools Comparison: E2E Encrypted.](/privacy-tools-guide/secure-file-sharing-tools-comparison/)
- [Best Encrypted File Sharing Service 2026](/privacy-tools-guide/best-encrypted-file-sharing-service-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
