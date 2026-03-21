---
layout: default
title: "Syncthing Setup Guide for Private File Sync"
description: "A practical guide to setting up Syncthing for secure, decentralized file synchronization between your devices. Perfect for developers and power users"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /syncthing-setup-guide-private-file-sync/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Syncthing is an open-source, decentralized file sync tool that transfers data directly between your devices using peer-to-peer connections, eliminating any cloud server and keeping your files under your complete control. Install Syncthing on macOS via Homebrew, Windows via installer, or Linux via package manager, then add devices by scanning QR codes or pasting device IDs and selecting folders to sync. This guide covers installation on all platforms, device pairing, folder configuration, security settings including TLS encryption and device verification, and practical setup for privacy-conscious file management across multiple devices.

## Installing Syncthing

Syncthing runs on Windows, macOS, Linux, and several other platforms. The installation methods vary depending on your operating system.

### macOS Installation

On macOS, you can install Syncthing using Homebrew:

```bash
brew install syncthing
```

After installation, launch Syncthing in the background:

```bash
syncthing serve &
```

### Linux Installation

Most Linux distributions package Syncthing in their repositories. On Debian or Ubuntu:

```bash
sudo apt install syncthing
```

For the latest version, download the binary directly from the GitHub releases:

```bash
wget https://github.com/syncthing/syncthing/releases/download/v1.27.2/syncthing-linux-amd64-v1.27.2.tar.gz
tar -xzf syncthing-linux-amd64-v1.27.2.tar.gz
./syncthing-linux-amd64-v1.27.2/syncthing
```

### Windows Installation

Download the Windows installer from the official website or use the portable version. The portable version works well for power users who prefer not to install software system-wide.

Verify your installation by accessing the web UI at `http://localhost:8384` after starting Syncthing.

## Initial Configuration

When you first run Syncthing, it generates a unique device ID and configuration files in your home directory. On Linux and macOS, these are stored in `~/.config/syncthing/`.

### Understanding the Web Interface

The web interface provides access to all Syncthing settings. Key sections include:

- **Folders**: Manage which directories are synchronized
- **Devices**: Add other devices to your sync network
- **Settings**: Configure global preferences including network, UI, and advanced options

### Adding Your First Folder

To synchronize a folder between devices:

1. Click "Add Folder" in the web interface
2. Enter a folder label (e.g., "Documents" or "Code Projects")
3. Specify the folder path on your local system
4. Configure sharing options by selecting which devices can access this folder

The folder path must exist on your system before Syncthing can sync it. Create the directory first if needed:

```bash
mkdir -p ~/Sync/Documents
```

## Connecting Devices

Syncthing identifies devices by their unique Device ID, a long string of characters generated during first startup. To connect two devices:

### Finding Your Device ID

Access the web interface and click the "Actions" menu (gear icon), then select "Show ID". Your Device ID will display along with a QR code for easy mobile device setup.

### Adding a Remote Device

On each device, navigate to "Add Device" and enter the other device's ID. Give the device a friendly name for easier identification. After adding the device, you can choose which folders to share with it.

### Network Configuration

Syncthing uses TCP port 22000 for local connections and can traverse NAT via UPnP or relaying through the Syncthing discovery servers. For better privacy, you may want to disable relay connections and discovery servers:

1. Go to Settings → Connections
2. Uncheck "Enable Relay Service"
3. Uncheck "Global Discovery" and "Local Discovery" if you prefer manual IP entry

For manual connections, specify the address directly:

```bash
tcp://192.168.1.100:22000
```

## Security Best Practices

Syncthing encrypts all transfers using TLS, but additional security measures strengthen your setup.

### Enabling HTTPS for Local Access

By default, Syncthing's web interface uses HTTP. Enable HTTPS for local access:

```bash
syncthing -gui-address=https://127.0.0.1:8384
```

### Configuring a GUI Password

Protect the web interface with authentication:

1. Navigate to Settings → GUI
2. Enable "GUI Authentication"
3. Set an username and password

For enhanced security, use environment variables to store credentials rather than storing them in configuration files:

```bash
SYNCTHING_GUI_USER=admin SYNCTHING_GUI_PASSWORD=your_secure_password syncthing
```

### Firewall Configuration

Ensure your firewall allows Syncthing's communication port:

```bash
# UFW example
sudo ufw allow 22000/tcp
```

## Advanced Configuration

For developers and power users, Syncthing offers advanced features through configuration files.

### Using the Config File

Edit `~/.config/syncthing/config.xml` directly for precise control. After editing, trigger a configuration reload:

```bash
syncthing -reload
```

### Ignoring Files

Create a `.stignore` file in each folder root to exclude files from synchronization:

```
# Ignore temporary files
*.tmp
*.swp
*~

# Ignore version control
.git/
.svn/

# Ignore OS files
.DS_Store
Thumbs.db
```

### Selective Synchronization

For large folders, use folder markers to synchronize only specific subdirectories. Create a `.stfolders` file listing the paths you want to sync:

```
Projects/webapp
Projects/mobile
Documents/Important
```

## Automating Syncthing Startup

### Systemd Service (Linux)

Create a systemd service for automatic startup:

```ini
[Unit]
Description=Syncthing - Open Source Continuous File Synchronization
After=network.target

[Service]
Type=simple
User=yourusername
ExecStart=/usr/bin/syncthing serve -no-browser -gui-address=http://127.0.0.1:8384
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl enable syncthing
sudo systemctl start syncthing
```

### LaunchAgent (macOS)

For macOS, create a LaunchAgent plist in `~/Library/LaunchAgents/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.github.xor-gate.syncthing</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/syncthing</string>
        <string>-no-browser</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

Load the agent:

```bash
launchctl load ~/Library/LaunchAgents/com.github.xor-gate.syncthing.plist
```

## Troubleshooting Common Issues

If devices won't connect, verify both have the correct Device ID added and that firewalls allow port 22000. At least one folder must be shared between them. If files aren't syncing, check the `.stignore` file for patterns that block them, then review the Syncthing log for specific error messages. High CPU usage can be reduced by lowering the concurrent file operation limit in Settings → Advanced → Max Concurrent Items.




## Related Reading

- [Best Self-Hosted File Sync Alternatives in 2026](/privacy-tools-guide/best-self-hosted-file-sync-alternative-2026/)
- [Encrypted File Sync for Teams Comparison: A Developer Guide](/privacy-tools-guide/encrypted-file-sync-for-teams-comparison/)
- [Privacy Tools For Private Investigator Protecting Case File](/privacy-tools-guide/privacy-tools-for-private-investigator-protecting-case-file-/)
- [Onionshare Secure File Sharing Over Tor Network Setup And Us](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Mumble Encrypted Voice Chat Server Setup For Private Team Co](/privacy-tools-guide/mumble-encrypted-voice-chat-server-setup-for-private-team-co/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
