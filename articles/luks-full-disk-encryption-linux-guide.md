---
layout: default
title: "LUKS Full Disk Encryption on Linux"
description: "How to set up LUKS2 full disk encryption on Linux from scratch, including partitioning, keyfile setup, crypttab configuration, and unlocking at boot"
date: 2026-03-21
author: theluckystrike
permalink: /luks-full-disk-encryption-linux-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

LUKS (Linux Unified Key Setup) is the standard for disk encryption on Linux. It wraps block devices in an encrypted container, protecting all data on the drive if it is physically stolen or accessed without authorization. This guide walks through setting up LUKS2 encryption on a new installation and on an existing secondary drive.

## LUKS2 vs LUKS1

LUKS2 is the current standard. Key improvements over LUKS1:

- **Argon2id key derivation function** — significantly stronger than LUKS1's PBKDF2 against brute force
- **Resilient header** with backup area — protects against header corruption
- **Larger metadata area** — supports multiple keyslots, tokens, and future features
- **JSON header** — machine-readable and extensible

Use LUKS2 for all new setups. Specify it explicitly since some tools still default to LUKS1.

## Prerequisites

```bash
sudo apt install cryptsetup cryptsetup-initramfs
```

Verify LUKS2 support:

```bash
cryptsetup --version
# Should show 2.x.x
```

## Option A: Encrypt a Secondary Drive

This is the simpler case — encrypting a data drive that does not contain your OS.

**Warning**: This destroys all existing data on the target drive.

```bash
# Identify the target drive (DO NOT use your system drive)
lsblk
# Example: /dev/sdb

# Create LUKS2 container with strong parameters
sudo cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  --pbkdf-memory 1048576 \
  --pbkdf-parallel 4 \
  --iter-time 5000 \
  /dev/sdb
```

You will be prompted to type YES (uppercase) and enter a passphrase. Use a strong passphrase — at least 6 random words from a wordlist.

```bash
# Open the encrypted container
sudo cryptsetup luksOpen /dev/sdb secure_data

# Create a filesystem inside it
sudo mkfs.ext4 /dev/mapper/secure_data

# Mount it
sudo mkdir -p /mnt/secure
sudo mount /dev/mapper/secure_data /mnt/secure
```

To close the container when done:

```bash
sudo umount /mnt/secure
sudo cryptsetup luksClose secure_data
```

## Option B: Full System Encryption During OS Install

Most Linux installers (Ubuntu, Debian, Fedora) offer full disk encryption during the setup wizard. Ubuntu calls it "Encrypt the new Ubuntu installation for security." This is the recommended path for a new system — the installer handles partition layout and bootloader integration automatically.

For manual control:

### Partition Layout

A typical LUKS-encrypted system needs:
- `/boot` partition — unencrypted (bootloader cannot decrypt LUKS at early boot on most systems)
- `/boot/efi` — EFI system partition, unencrypted
- LUKS container — contains LVM or direct partitions for `/`, `swap`, `/home`

```bash
# Partition the drive (adjust sizes to your needs)
sudo parted /dev/sda -- mklabel gpt
sudo parted /dev/sda -- mkpart ESP fat32 1MiB 512MiB
sudo parted /dev/sda -- set 1 esp on
sudo parted /dev/sda -- mkpart boot ext4 512MiB 1GiB
sudo parted /dev/sda -- mkpart primary 1GiB 100%

# Format EFI and boot
sudo mkfs.fat -F32 /dev/sda1
sudo mkfs.ext4 /dev/sda2

# Create LUKS on the main partition
sudo cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/sda3

sudo cryptsetup luksOpen /dev/sda3 cryptroot

# Set up LVM inside LUKS
sudo pvcreate /dev/mapper/cryptroot
sudo vgcreate vg0 /dev/mapper/cryptroot
sudo lvcreate -L 16G vg0 -n swap
sudo lvcreate -l 100%FREE vg0 -n root

sudo mkswap /dev/vg0/swap
sudo mkfs.ext4 /dev/vg0/root
```

## Configuring Auto-Mount via crypttab

To have the system prompt for a LUKS passphrase at boot and mount the drive automatically, add an entry to `/etc/crypttab`:

```bash
# Get the UUID of the encrypted partition
sudo blkid /dev/sdb | grep UUID
# Example output: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890"

# Add to /etc/crypttab
echo "secure_data UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890 none luks" \
  | sudo tee -a /etc/crypttab

# Add to /etc/fstab for auto-mount
echo "/dev/mapper/secure_data /mnt/secure ext4 defaults 0 2" \
  | sudo tee -a /etc/fstab
```

After rebooting, the system will prompt for the passphrase before mounting the drive.

## Adding a Keyfile (for Convenience)

A keyfile allows unlocking the container without typing a passphrase — useful for automatically mounting secondary drives while still keeping your root partition passphrase-protected.

```bash
# Generate a strong keyfile
sudo dd if=/dev/urandom of=/etc/luks-keys/secure_data.key bs=4096 count=1
sudo chmod 600 /etc/luks-keys/secure_data.key

# Add keyfile as an additional key slot
sudo cryptsetup luksAddKey /dev/sdb /etc/luks-keys/secure_data.key

# Update crypttab to use the keyfile
# Replace "none" with the keyfile path
sudo nano /etc/crypttab
# secure_data UUID=... /etc/luks-keys/secure_data.key luks
```

Keep the passphrase as a backup key slot — if the keyfile is lost or the system disk fails, you can still unlock the drive with the passphrase.

## Inspecting and Managing Key Slots

LUKS supports up to 32 key slots (LUKS2). You can have multiple passphrases, keyfiles, or recovery keys:

```bash
# Show LUKS header information including active key slots
sudo cryptsetup luksDump /dev/sdb

# Add a backup passphrase (e.g., for recovery)
sudo cryptsetup luksAddKey /dev/sdb

# Remove a specific key slot (be careful — ensure another slot works first)
sudo cryptsetup luksKillSlot /dev/sdb 1
```

## Benchmarking Your Configuration

Test encryption speed to ensure your parameters are practical:

```bash
# Benchmark available cipher options
sudo cryptsetup benchmark

# Test the key derivation time for your LUKS header
sudo cryptsetup luksDump /dev/sdb | grep -A5 "Key Slot 0"
```

The Argon2id settings above target ~5 seconds for key derivation — slow enough to resist brute force but fast enough not to be annoying at boot.

## Backup the LUKS Header

The LUKS header contains the key slots. If it is corrupted, all data is unrecoverable. Back it up:

```bash
sudo cryptsetup luksHeaderBackup /dev/sdb \
  --header-backup-file /secure/luks-header-backup-sdb.img
```

Store this backup encrypted (ironically, in another LUKS container or encrypted ZIP) and off-device.

## Related Reading

- [VeraCrypt Full Disk Encryption Setup Guide](/veracrypt-full-disk-encryption-setup-guide/)
- [Secure File Deletion on SSD Drives](/secure-file-deletion-ssd-drives-guide/)
- [Air-Gapped Computer Setup for Maximum Security](/air-gapped-computer-setup-for-maximum-security-practical-gui/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
