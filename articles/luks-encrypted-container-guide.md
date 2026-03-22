---
layout: default
title: "How to Create an Encrypted Container with LUKS"
description: "Create portable LUKS encrypted containers as loop device files on Linux, separate from full-disk encryption, for encrypting specific directories and files"
date: 2026-03-22
author: theluckystrike
permalink: /luks-encrypted-container-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Create an Encrypted Container with LUKS

LUKS full-disk encryption protects everything on a disk. But sometimes you need a portable, specific encrypted container — a single file that acts like an encrypted vault you can move around, back up, or mount on any Linux system. This is done using a loop device: LUKS on a regular file.

This guide walks through creating a LUKS2 container from scratch, including choosing the right cipher and key derivation function, managing keyslots, backing up the header, resizing the container, and using convenience scripts for daily use.

---

## Why a Container File vs Full Disk Encryption

| Use case | Full disk | Container file |
|----------|-----------|----------------|
| Entire system | Best | Not suitable |
| Specific files | Overkill | Ideal |
| Portable vault | Inconvenient | Copy the file |
| Backup-friendly | Requires block-level backup | Just copy the file |
| Works alongside unencrypted files | No | Yes |

The container file approach is what VeraCrypt calls an "outer volume." The difference is that LUKS is Linux-native and uses the kernel's dm-crypt layer directly, which means it integrates cleanly with standard Linux tools, has no proprietary components, and performs at near-native speed because encryption/decryption happens in the kernel.

---

## Create a Container File

```bash
# Create a 2GB container file filled with random data
# Random fill hides the size of actual data inside
sudo dd if=/dev/urandom of=/home/user/vault.luks bs=1M count=2048 status=progress

# Faster alternative using openssl rand
openssl rand -out /home/user/vault.luks $((2 * 1024 * 1024 * 1024))
```

Filling the container with random data before formatting is important for two reasons. First, it prevents an attacker from telling how much of the container is actually used — an empty LUKS container with no random fill has an obvious pattern of zeros in unused sectors, which leaks information about how full your vault is. Second, it provides no foothold for statistical attacks against the encrypted data.

The `dd if=/dev/urandom` approach is slower because it reads from the kernel's random device. On a machine without hardware random number generation (RDRAND/RDSEED), `/dev/urandom` may not generate data fast enough to keep up with disk write speed. In that case, use `openssl rand` which uses OpenSSL's PRNG seeded from the kernel entropy pool — it produces cryptographically indistinguishable output and is much faster.

To monitor the `dd` progress with the `status=progress` flag, you will see throughput and estimated completion time in real time. For a 10GB container on a modern SSD, expect roughly 2-3 minutes.

---

## Initialize LUKS on the Container

```bash
# Set up LUKS2 on the container file
sudo cryptsetup luksFormat --type luks2 /home/user/vault.luks

# Confirm: type "YES" (uppercase) and enter a strong passphrase

# View the LUKS header details
sudo cryptsetup luksDump /home/user/vault.luks
```

LUKS2 is the current format and should be preferred over LUKS1. Key differences: LUKS2 uses Argon2id for key derivation (LUKS1 used PBKDF2), supports up to 32 keyslots (LUKS1 had 8), and stores the header in two locations with checksums to detect corruption. The header is 16 MB rather than 1-2 MB, so your container file needs a bit of extra headroom.

By default, `luksFormat` uses AES-256-XTS with SHA-256. To use a stronger configuration:

```bash
sudo cryptsetup luksFormat \
    --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --pbkdf argon2id \
    --pbkdf-memory 524288 \
    --pbkdf-parallel 4 \
    /home/user/vault.luks
```

The `--key-size 512` gives you AES-256 in XTS mode (XTS splits the 512-bit key into two 256-bit halves). `--pbkdf-memory 524288` sets Argon2id's memory cost to 512 MB — this makes brute-force attacks on your passphrase expensive because each attempt requires 512 MB of RAM. Calibrate this to your machine: `cryptsetup benchmark --pbkdf argon2id` shows what your hardware can handle.

---

## Open and Format the Container

```bash
# Map the container to a device name
sudo cryptsetup luksOpen /home/user/vault.luks myvault
# Enter passphrase → creates /dev/mapper/myvault

# Create a filesystem inside
sudo mkfs.ext4 /dev/mapper/myvault
# For cross-platform: sudo mkfs.exfat /dev/mapper/myvault

# Mount it
sudo mkdir -p /mnt/vault
sudo mount /dev/mapper/myvault /mnt/vault
sudo chown -R $USER:$USER /mnt/vault

df -h /mnt/vault
```

After `cryptsetup luksOpen`, the device `/dev/mapper/myvault` appears as a standard block device. From this point forward, the kernel handles transparent encryption and decryption — any read or write to `/dev/mapper/myvault` goes through AES-XTS automatically.

Choose the filesystem based on your use case. `ext4` is the right choice for Linux-only vaults — it supports extended attributes, Unix permissions, and large files without overhead. `exfat` is appropriate if you need to open the vault on macOS or Windows (with the appropriate cryptsetup port or utility), but it lacks journaling and Unix permission support. `btrfs` is worth considering for vaults where you want snapshots or transparent compression.

When formatting with `mkfs.ext4`, add a label for easier identification:

```bash
sudo mkfs.ext4 -L "myvault" /dev/mapper/myvault
```

The label appears in `lsblk -o NAME,LABEL,SIZE` output, making it easy to confirm you are operating on the right device when you have multiple containers open simultaneously.

---

## Close the Container

```bash
# Unmount the filesystem first
sudo umount /mnt/vault

# Close the LUKS mapping (re-encrypts and locks)
sudo cryptsetup luksClose myvault
```

After `luksClose`, the `/dev/mapper/myvault` device disappears. The `vault.luks` file is now a sealed, encrypted blob. Any running process that had open file descriptors to the mount point will get I/O errors after unmount — close all applications using the vault before running `luksClose`.

---

## Convenience Scripts

```bash
# /usr/local/bin/vault-open
#!/bin/bash
set -e
CONTAINER="/home/$USER/vault.luks"
MOUNT="/mnt/vault"
NAME="myvault"

if cryptsetup status $NAME &>/dev/null; then
    echo "Vault already open at $MOUNT"
    exit 0
fi

sudo cryptsetup luksOpen "$CONTAINER" "$NAME"
sudo mount /dev/mapper/$NAME "$MOUNT"
sudo chown -R $USER:$USER "$MOUNT"
echo "Vault open at $MOUNT"
```

```bash
# /usr/local/bin/vault-close
#!/bin/bash
sudo umount /mnt/vault && \
sudo cryptsetup luksClose myvault && \
echo "Vault locked."
```

Make both scripts executable and owned by root, with the SUID bit or sudo rules to allow the current user to run them without a password:

```bash
sudo cp vault-open vault-close /usr/local/bin/
sudo chmod +x /usr/local/bin/vault-open /usr/local/bin/vault-close
```

Add these two lines to `/etc/sudoers` via `visudo` to allow passwordless execution:

```
your_username ALL=(root) NOPASSWD: /usr/bin/cryptsetup, /usr/bin/mount, /usr/bin/umount, /usr/bin/chown
```

---

## Add Additional Passphrases (Keyslots)

LUKS2 supports up to 32 keyslots. You can add a second passphrase for backup:

```bash
# Add a second passphrase to keyslot 1
sudo cryptsetup luksAddKey /home/user/vault.luks

# Use a keyfile for automation (no interactive passphrase)
dd if=/dev/urandom of=/root/vault.keyfile bs=512 count=1
chmod 400 /root/vault.keyfile

sudo cryptsetup luksAddKey /home/user/vault.luks /root/vault.keyfile

# Open with keyfile
sudo cryptsetup luksOpen --key-file=/root/vault.keyfile /home/user/vault.luks myvault
```

Store a printed copy of the second passphrase in a physically secure location, separate from the vault file. If you forget your primary passphrase and have no backup, the data is permanently unrecoverable — LUKS encryption has no backdoor.

---

## Backup the LUKS Header

The LUKS header is at the beginning of the container file. If corrupted, all data is permanently lost.

```bash
# Backup the header
sudo cryptsetup luksHeaderBackup /home/user/vault.luks \
    --header-backup-file /home/user/vault.luks.header.bak

# Store this backup separately from the container

# Restore from backup if header is corrupted
sudo cryptsetup luksHeaderRestore /home/user/vault.luks \
    --header-backup-file /home/user/vault.luks.header.bak
```

The header backup is small (16 MB for LUKS2) and contains the encryption parameters and all keyslots, but not the actual key material — without the passphrase, the header backup is useless to an attacker. Store it on a different physical medium than the container file itself. If both are on the same drive and that drive fails, you lose access regardless of the header backup.

After any `luksAddKey` or `luksChangeKey` operation, re-export the header backup since keyslot data has changed.

---

## Resize the Container

```bash
# To grow the container:
vault-close

# Add space to the container file
dd if=/dev/urandom bs=1M count=512 >> /home/user/vault.luks

# Reopen and resize
sudo cryptsetup luksOpen /home/user/vault.luks myvault
sudo resize2fs /dev/mapper/myvault

# Verify
sudo mount /dev/mapper/myvault /mnt/vault && df -h /mnt/vault
```

Shrinking a LUKS container is more complex and risky — you must shrink the filesystem first (`resize2fs /dev/mapper/myvault SIZE`), then close the container, then truncate the container file. Shrinking incorrectly destroys data. Growing is safe and straightforward.

---

## Portable Use

The `vault.luks` file can be:
- Copied to a USB drive and opened on any Linux system
- Backed up to encrypted cloud storage
- Transferred via OnionShare

```bash
# On another Linux machine:
sudo cryptsetup luksOpen vault.luks myvault
sudo mount /dev/mapper/myvault /mnt/vault
# Enter passphrase — works on any Linux with cryptsetup installed
```

---

## Related Reading

- [LUKS Full Disk Encryption Linux Guide](/privacy-tools-guide/luks-full-disk-encryption-linux-guide/)
- [Secure Boot and TPM Explained for Linux](/privacy-tools-guide/secure-boot-tpm-linux-explained/)
- [Encrypt USB Drive with VeraCrypt](/privacy-tools-guide/encrypt-usb-drive-veracrypt-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
