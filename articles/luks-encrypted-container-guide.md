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

---

## Why a Container File vs Full Disk Encryption

| Use case | Full disk | Container file |
|----------|-----------|----------------|
| Entire system | Best | Not suitable |
| Specific files | Overkill | Ideal |
| Portable vault | Inconvenient | Copy the file |
| Backup-friendly | Requires block-level backup | Just copy the file |
| Works alongside unencrypted files | No | Yes |

---

## Create a Container File

```bash
# Create a 2GB container file filled with random data
# Random fill hides the size of actual data inside
sudo dd if=/dev/urandom of=/home/user/vault.luks bs=1M count=2048 status=progress

# Faster alternative using openssl rand
openssl rand -out /home/user/vault.luks $((2 * 1024 * 1024 * 1024))
```

---

## Initialize LUKS on the Container

```bash
# Set up LUKS2 on the container file
sudo cryptsetup luksFormat --type luks2 /home/user/vault.luks

# Confirm: type "YES" (uppercase) and enter a strong passphrase

# View the LUKS header details
sudo cryptsetup luksDump /home/user/vault.luks
```

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

---

## Close the Container

```bash
# Unmount the filesystem first
sudo umount /mnt/vault

# Close the LUKS mapping (re-encrypts and locks)
sudo cryptsetup luksClose myvault
```

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
