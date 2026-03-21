---
layout: default
title: "Tails Persistent Storage Setup Guide What To Save And What S"
description: "A practical guide for developers and power users on setting up Tails persistent storage, understanding what data persists between sessions, and what"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /tails-persistent-storage-setup-guide-what-to-save-and-what-s/
categories: [guides, security]
reviewed: true
score: 9
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

## Advanced Persistent Storage Encryption

For sensitive files in persistent storage, apply additional encryption layers:

```bash
# Create encrypted vault within persistent storage
mkdir -p /home/amnesia/Persistent/.vaults

# Generate random encryption key
openssl rand -base64 32 > /home/amnesia/Persistent/.vault-key

# Create encrypted volume with cryptsetup
sudo cryptsetup luksFormat --type luks2 /home/amnesia/Persistent/.vaults/sensitive.img

# Mount encrypted volume
sudo cryptsetup luksOpen /home/amnesia/Persistent/.vaults/sensitive.img vault
sudo mount /dev/mapper/vault /mnt/vault

# Unmount and close when done
sudo umount /mnt/vault
sudo cryptsetup luksClose vault
```

This nested encryption ensures that even if someone accesses your persistent storage, the most sensitive data remains protected.

## Persistent Storage File Organization Best Practices

Structure your persistent directory for efficiency and security:

```
/home/amnesia/Persistent/
├── .config/           # Application configurations (hidden)
├── .ssh/              # SSH keys (hidden)
├── .gnupg/            # GPG keys (hidden)
├── projects/          # Work files
│   ├── client-a/
│   └── client-b/
├── documents/         # Writing and notes
├── backups/           # Encrypted backups
└── temp/              # Temporary files (not critical)
```

Keep sensitive files in hidden directories (prefixed with dot) and use restrictive permissions:

```bash
# Restrict access to sensitive directories
chmod 700 /home/amnesia/Persistent/.ssh
chmod 700 /home/amnesia/Persistent/.gnupg

# Verify permissions
ls -la /home/amnesia/Persistent/
```

## Monitoring Persistent Storage Usage

Track how much persistent storage you're using to avoid filling the USB drive:

```bash
#!/bin/bash
# Monitor persistent storage space
echo "=== Persistent Storage Analysis ==="

# Show total and used space
du -sh /home/amnesia/Persistent

# Show breakdown by directory
du -sh /home/amnesia/Persistent/* | sort -rh | head -10

# Show what changed in last 7 days
find /home/amnesia/Persistent -type f -mtime -7 -exec ls -lh {} \;

# Estimate storage efficiency
total=$(du -sb /home/amnesia/Persistent | cut -f1)
largest=$(du -sb /home/amnesia/Persistent | cut -f1)
echo "Total storage used: $((total / 1024 / 1024))MB"
```

## Threat Models for Persistent Storage Configuration

Different threat levels require different strategies:

```python
threat_models = {
    "low_threat": {
        "scenario": "General privacy, casual Tor browsing",
        "persist": [
            "SSH keys for personal servers",
            "Browser bookmarks",
            "Project files"
        ],
        "ephemeral": [
            "Browser history",
            "Cache",
            "Temporary work files"
        ]
    },
    "medium_threat": {
        "scenario": "Journalist, activist in moderately hostile environment",
        "persist": [
            "GPG keys (encrypted)",
            "SSH keys (encrypted)",
            "Important documents (encrypted)"
        ],
        "ephemeral": [
            "All communications",
            "Research notes",
            "Contact information"
        ]
    },
    "high_threat": {
        "scenario": "Dissident in repressive regime, targeted by state actors",
        "persist": [
            "Only absolutely critical keys",
            "All data deeply encrypted"
        ],
        "ephemeral": [
            "Everything except essential keys",
            "Rotate USB keys frequently",
            "Use separate USB for separate personas"
        ]
    }
}
```

## Backup Strategy for Persistent Storage

While Tails provides no internet by default, backups of persistent storage are essential:

```bash
# Create encrypted backup of persistent storage
# Use only on external USB drive in secure location

# 1. Insert external USB drive
# 2. Mount it securely
sudo mount /media/backup-drive

# 3. Create encrypted tar backup
tar --exclude='.*' \
    --exclude='temp/*' \
    -czf - /home/amnesia/Persistent | \
    openssl enc -aes-256-cbc -salt \
    > /media/backup-drive/tails-persistent-backup-$(date +%Y%m%d).tar.gz.enc

# 4. Verify backup size
ls -lh /media/backup-drive/*.enc

# 5. Securely unmount
sudo umount /media/backup-drive
```

To restore from backup:

```bash
# Decrypt and extract backup
openssl enc -aes-256-cbc -d -salt \
    -in /media/backup-drive/tails-persistent-backup-20260316.tar.gz.enc | \
    tar -xzf - -C /home/amnesia/
```

## Recovery Scenarios and Persistent Storage

Understanding recovery options helps inform your persistent storage decisions:

```python
# Recovery scenarios and implications
recovery_scenarios = {
    "lost_passphrase": {
        "status": "Permanent loss of data",
        "prevention": "Store passphrase in password manager with offline backup",
        "recovery": "None—USB becomes unusable"
    },
    "corrupted_persistent_volume": {
        "status": "Potential data loss",
        "prevention": "Regular encrypted backups",
        "recovery": "Restore from backup if available"
    },
    "device_seized": {
        "status": "Depends on encryption and threat model",
        "prevention": "Never carry sensitive data across borders",
        "recovery": "Device is lost; focus on other devices"
    },
    "accidental_deletion": {
        "status": "Data lost unless encrypted backup exists",
        "prevention": "Regular backups, immutable storage",
        "recovery": "Restore from recent backup"
    }
}
```

## Persistent Storage Performance Optimization

Encrypted persistent storage is slower than regular storage. Optimize performance:

```bash
# Move frequently-accessed files to RAM tmpfs
sudo mount -t tmpfs -o size=512M tmpfs /mnt/ramdisk

# Copy working directory to RAM
cp -r /home/amnesia/Persistent/projects /mnt/ramdisk/

# Work from RAM, then copy back
cp -r /mnt/ramdisk/projects /home/amnesia/Persistent/

# This improves responsiveness while keeping persistent copy encrypted
```

Understanding what persists and what disappears is fundamental to using Tails effectively. By strategically selecting what survives reboots, you maintain both operational convenience and the strong anonymity guarantees that make Tails valuable.



## Related Reading

- [How to Use Tails OS for Maximum Privacy Complete Setup Guide](/privacy-tools-guide/how-to-use-tails-os-for-maximum-privacy-complete-setup-guide/)
- [Nextcloud External Storage Setup Guide 2026](/privacy-tools-guide/nextcloud-external-storage-setup-guide-2026/)
- [WireGuard Persistent Keepalive Setting Explained](/privacy-tools-guide/wireguard-persistent-keepalive-setting-explained-when-to-enable-it/)
- [How to Use Tails Operating System for Extreme Privacy Daily](/privacy-tools-guide/how-to-use-tails-operating-system-for-extreme-privacy-daily/)
- [How to Use Tails OS for Maximum Anonymity](/privacy-tools-guide/how-to-use-tails-os-for-maximum-anonymity/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
