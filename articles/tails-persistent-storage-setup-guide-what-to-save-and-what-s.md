---
layout: default
title: "Tails Persistent Storage Setup Guide What To Save And What S"
description: "A practical guide for developers and power users on setting up Tails persistent storage, understanding what data persists between sessions, and what."
date: 2026-03-16
author: theluckystrike
permalink: /tails-persistent-storage-setup-guide-what-to-save-and-what-s/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Tails is a privacy-focused operating system that runs from an USB stick, routing all traffic through the Tor network. By default, Tails leaves no trace on the computer you use—but this anonymity comes with a trade-off: every shutdown wipes your session clean. The persistent storage feature solves this problem by creating an encrypted volume on your USB drive that survives reboots. This guide explains how to set it up, what belongs in persistent storage, and what data should remain ephemeral for maximum security.

## How Persistent Storage Works in Tails

When you activate persistent storage, Tails creates an encrypted partition alongside the operating system on your USB stick. This partition uses LUKS (Linux Unified Key Setup) encryption, protected by a passphrase you choose during setup. The encryption key is derived from your passphrase using PBKDF2, making it resistant to brute-force attacks.

The persistent volume mounts automatically when you enter your passphrase at the Tails welcome screen. Files you save to specific directories (`/home/amnesia/Persistent` and others you configure) persist across sessions. Everything else—browser history, temporary files, system logs—disappears when you shut down.

To verify persistent storage is available on your Tails USB, check for the "Persistent" folder in your home directory:

```bash
ls -la /home/amnesia/Persistent
```

If the folder exists and contains your files after a restart, your persistent storage is working correctly.

## Setting Up Persistent Storage

Setting up persistent storage requires administrative access within Tails and takes approximately 10-15 minutes. Follow these steps:

1. Boot into Tails and select your language and keyboard layout
2. From the desktop, open "Applications" → "Utilities" → "Configure Persistent Volume"
3. Enter a strong passphrase (use a password manager to generate 20+ random characters)
4. Select which directories to persist—you can choose from预设选项

The configuration tool offers these persistent directory options:

- **Persistent**: The main folder for your files, accessible via `/home/amnesia/Persistent`
- **GnuPG keys**: PGP private keys for encrypted communication
- **SSH keys**: SSH authentication keys for remote server access
- **Claws Mail**: Email client data and stored messages
- **Thunderbird**: Mozilla Thunderbird profile and emails
- **Pidgin**: Instant messaging configuration and logs
- **Web cookies**: Session cookies for websites (use with caution)
- **Browser bookmarks**: Saved bookmarks from the Tor Browser
- **Network connections**: Saved Wi-Fi credentials and network configurations

Enable only the directories you need. Each enabled directory increases your attack surface if the USB is compromised.

## What Belongs in Persistent Storage

Certain data should survive reboots because recreating it manually each session is impractical or impossible. For developers and power users, these categories make sense in persistent storage:

### GnuPG and SSH Keys

Your PGP keys and SSH credentials must persist—without them, you cannot decrypt messages or authenticate to servers. Store them in their respective persistent folders:

```bash
# Verify GnuPG key persistence
gpg --list-secret-keys

# Check SSH agent for loaded keys
ssh-add -l
```

Regenerating PGP keys frequently raises suspicion and breaks existing trust relationships. Keep your master keys in persistent storage with backups in a secure location.

### Development Environment Configuration

Dotfiles, shell configurations, and development tool settings belong in persistent storage:

```bash
# Common persistent configuration paths
/home/amnesia/Persistent/dotfiles/
/home/amnesia/Persistent/.config/
/home/amnesia/Persistent/.ssh/
```

Create symlinks from your home directory to the persistent versions:

```bash
ln -s /home/amnesia/Persistent/dotfiles/.bashrc ~/.bashrc
ln -s /home/amnesia/Persistent/dotfiles/.gitconfig ~/.gitconfig
```

### Encrypted Password Databases

If you use a password manager like Bitwarden, KeePassXC, or 1Password, store the encrypted vault in persistent storage. The encryption protects the data even if someone obtains your USB stick:

```bash
# Example: KeePassXC database location
/home/amnesia/Persistent/keepass/Database.kdbx
```

### Source Code and Work Files

Active development projects should persist between sessions. Clone repositories to the persistent directory:

```bash
cd /home/amnesia/Persistent
git clone git@github.com:yourusername/project.git
```

## What Must Remain Ephemeral

Some data should never persist for security reasons. Understanding what Tails intentionally wipes protects your anonymity.

### Browser Session Data

All browser data—history, cookies, cache, and local storage—should remain ephemeral. This prevents correlation attacks where an adversary analyzes your browsing patterns across sessions. The Tor Browser isolates tabs and clears data on exit by design; enabling persistent cookies defeats this protection.

### Temporary Files and Clipboard

Any file in `/tmp` or `/var/tmp` gets wiped automatically. The clipboard contents also clear on shutdown. If you're copying sensitive data, expect it to disappear when you reboot.

### System Logs

Tails does not write persistent system logs for good reason—logs contain timestamps, connection metadata, and system events that could compromise your anonymity. Do not redirect logs to persistent storage.

### Swap Data

Linux swap partitions can contain sensitive data from memory. Tails disables swap by default, but verify this setting if using advanced configurations:

```bash
# Check swap status
swapon --show
```

If swap is active, disable it: `swapoff -a`

## Security Trade-offs and Best Practices

Persistent storage creates a trade-off between convenience and security. The more data you persist, the more information exists for an adversary to analyze if your USB is seized or compromised.

Follow these hardening practices:

1. **Use separate USB drives** for different personas or activities
2. **Encrypt sensitive files** within persistent storage using GPG or age
3. **Rotate passphrases** periodically, especially after crossing borders
4. **Enable the "Eraser" feature** in Tails to securely wipe unused persistent space
5. **Never store plaintext credentials**—always use encrypted vaults

For additional protection, consider using LUKS metadata randomization or plausible deniability tools, though these advanced techniques require careful configuration.

Understanding what persists and what disappears is fundamental to using Tails effectively. By strategically selecting what survives reboots, you maintain both operational convenience and the strong anonymity guarantees that make Tails valuable.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
