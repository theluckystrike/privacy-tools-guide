---

layout: default
title: "Syncthing Setup Guide: Private File Sync for Developers"
description: "A practical guide to setting up Syncthing for encrypted, decentralized file synchronization. Perfect for developers and power users who need secure, serverless file sync across their devices."
date: 2026-03-15
author: theluckystrike
permalink: /syncthing-setup-guide-private-file-sync/
categories: [guides]
---

{% raw %}
Syncthing is an open-source, decentralized file synchronization tool that replaces proprietary cloud storage with something you control entirely. Unlike Dropbox or Google Drive, no third-party servers store your data. Instead, devices communicate directly using encrypted connections.

This guide covers installing Syncthing, pairing devices, configuring folders, and hardening security for developers and power users who value privacy and self-hosted solutions.

## Why Syncthing?

Syncthing offers several advantages over traditional cloud sync tools:

- **No account required** — No email signup, no cloud provider, no vendor lock-in
- **End-to-end encryption** — Data is encrypted before leaving your device
- **Decentralized** — Devices connect peer-to-peer; no intermediate servers
- **Open source** — Audit the code, contribute, or fork it
- **Cross-platform** — Runs on Linux, Windows, macOS, Android, and BSD

For developers managing code, configs, or sensitive documents across machines, Syncthing provides a reliable alternative to git-based sync or USB drives.

## Installing Syncthing

Syncthing runs as a background service with a web-based management interface. Installation varies by platform.

### Linux

On Debian-based systems:

```bash
sudo apt install syncthing
```

On Arch Linux:

```bash
sudo pacman -S syncthing
```

Start it manually:

```bash
syncthing serve
```

Or enable as a systemd service for automatic startup:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/syncthing.service <<EOF
[Unit]
Description=Syncthing - Open Source File Synchronization

[Service]
ExecStart=/usr/bin/syncthing serve --no-browser --no-restart --logflags=0
Restart=on-failure

[Install]
WantedBy=default.target
EOF

systemctl --user enable syncthing
systemctl --user start syncthing
```

### macOS

Install via Homebrew:

```bash
brew install syncthing
```

Launch at login:

```bash
brew services start syncthing
```

### Windows

Download the executable from [syncthing.net](https://syncthing.net). Run `syncthing.exe` — it starts minimized to the system tray with a web interface at `http://localhost:8384`.

## Initial Configuration

Open the web interface at `http://localhost:8384` (or the IP of your headless server on port 8384). The default settings work, but several changes improve security.

### Change Default Port and Disable Discovery

In the web UI, navigate to **Actions → Settings → General**:

1. Set a custom GUI port (e.g., 18474 instead of 8384)
2. Enable HTTPS for the GUI (generate a self-signed certificate or use your own)
3. Disable global discovery if you only want local network connections
4. Disable NAT traversal unless you need穿越 NAT

These settings reduce attack surface and prevent unexpected external connections.

### Set a GUI Authentication User

Under **Settings → GUI**, enable authentication. Use a strong username and password:

```bash
# Generate a bcrypt hash for your password (requires Python)
python3 -c "import bcrypt; print(bcrypt.hashpw(b'your_password_here', bcrypt.gensalt()).decode())"
```

Add credentials to `~/.config/syncthing/config.xml`:

```xml
<gui enabled="true" tls="true">
  <address>127.0.0.1:18474</address>
  <user>your_username</user>
  <password>your_bcrypt_hash</password>
</gui>
```

## Pairing Devices

Syncthing uses device IDs rather than accounts. Each installation generates a unique 64-character device ID.

### Finding Your Device ID

In the web UI, click the device icon in the bottom-left corner. Your device ID displays there. Copy it — you'll share this with other devices you want to sync.

### Adding a Remote Device

On the receiving device:

1. Click **Add Remote Device** in the web UI
2. Paste the other device's device ID
3. Assign a friendly name (e.g., "Laptop", "Desktop")
4. Choose which folders to share with this device
5. Click **Save**

The initiating device receives a connection request. Approve it on both ends to establish the connection.

### Device ID Verification

For security, verify that the device ID matches what you expect. Man-in-the-middle attacks are theoretically possible during initial pairing. Check the fingerprint in **Actions → Device → Show ID** on each device and confirm they match.

## Configuring Folders

Folders in Syncthing map local directories to sync targets. Create a new folder:

1. Click **Add Folder** in the web UI
2. Specify the folder label and path
3. Choose sharing settings (which devices get this folder)
4. Configure versioning if needed

### Folder Types

- **Send & Receive** — Bidirectional sync; changes propagate both ways
- **Send Only** — Useful for distributing files to other devices without receiving their changes
- **Receive Only** — Pull files from other devices but don't send local changes

### File Versioning

Enable versioning to protect against accidental deletion or overwrite:

```xml
<folder id="xyz" path="/home/user/Sync" type="sendreceive" versioner="trashcan">
  <versioning type="trashcan" />
</folder>
```

The `trashcan` versioner moves old versions to a `.syncthing-versions` subdirectory. Configure retention settings in the web UI.

### Ignoring Files

Use `.stignore` patterns to exclude files from sync. Common patterns:

```
# Ignore hidden files
.*
# Ignore specific extensions
*.tmp
*.log
# Ignore node_modules
node_modules/
# Except keep .env (important!)
!.env
```

Place `.stignore` in the folder root. Changes take effect after a rescan.

## Network Configuration

Syncthing uses TCP and UDP ports 22000/21027 by default. For devices on different networks, configure port forwarding on your router:

- **TCP 22000** — Main sync protocol
- **UDP 21027** — Local discovery

If you cannot open ports (common with CGNAT or corporate networks), enable **relay servers** in Settings → Connections. Syncthing includes free public relays, though they add latency.

For self-hosted relay, run your own relay server:

```bash
docker run -p 19267:19267 -d syncthing/relaysrv \
  -keys=/keys -relays=20 \
  -provided-by="Your Name"
```

## Security Hardening

### Enable TLS Everywhere

Ensure all GUI and sync connections use TLS. In Settings → Connections, check **Force TLS**.

### Use a Firewall

Restrict access to the GUI port:

```bash
# Allow only localhost
sudo ufw deny 18474/tcp

# Or allow specific IP
sudo ufw allow from 192.168.1.100 to any port 18474
```

### Monitor Connections

Check the **Status** page in the web UI for active connections and transfer rates. Unexpected connections warrant investigation.

## Automating with the CLI

Syncthing includes a command-line tool. Use it for scripting:

```bash
# Check status
syncthing cli config devices

# Trigger a rescan
syncthing cli internal rescan /path/to/folder

# Get connection status
syncthing cli status
```

Integrate Syncthing into your workflow with cron jobs or systemd timers for scheduled folder scans.

## Troubleshooting

- **Devices won't connect** — Check firewall rules, ensure ports are open, verify device IDs
- **Slow sync** — Test direct connection speeds; consider relay if NAT blocks direct access
- **High CPU** — Reduce max folder concurrency in Settings → Advanced
- **Conflicts** — Syncthing names conflict copies with timestamps; resolve manually

## Wrapping Up

Syncthing provides a privacy-first alternative to cloud storage with full user control. By running it on your own devices — a home server, always-on desktop, or VPS — you maintain ownership of your data without sacrificing convenience.

Start with one folder between two devices. Add more folders, devices, and automation as needs grow. The learning curve is minimal, and the flexibility rewards power users who want control over their sync infrastructure.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
