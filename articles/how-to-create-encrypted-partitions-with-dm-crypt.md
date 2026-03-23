---
layout: default
title: "How to Create Encrypted Partitions with dm-crypt"
description: "Encrypt Linux disk partitions with LUKS and dm-crypt using AES-256 encryption, keyfiles, and automatic unlocking with TPM or USB tokens"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-encrypted-partitions-with-dm-crypt/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

dm-crypt is the Linux kernel's device mapper encryption subsystem. LUKS (Linux Unified Key Setup) is the standard on-disk format for LUKS-encrypted partitions. it handles key management, metadata, and up to 8 key slots (passphrases or keyfiles). This guide covers creating encrypted partitions for data drives, adding keyfiles, and setting up automatic mounting.

Prerequisites

```bash
Install cryptsetup
sudo apt install cryptsetup

Verify kernel module is loaded
lsmod | grep dm_crypt
If empty:
sudo modprobe dm_crypt
```

Step 1: Identify Your Target Disk

```bash
List block devices
lsblk -f

More detail
sudo fdisk -l

Identify the disk you want to encrypt (e.g., /dev/sdb)
All data on this device will be destroyed
```

For this guide, the target device is `/dev/sdb`. Replace with your actual device path.

Step 2: Create a LUKS Container

```bash
Format the device with LUKS2 (default in modern cryptsetup)
Uses AES-256-XTS (XTS mode is standard for disk encryption)
sudo cryptsetup luksFormat /dev/sdb

Or specify parameters explicitly
sudo cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha256 \
  --pbkdf argon2id \
  --pbkdf-memory 131072 \
  --pbkdf-parallel 4 \
  /dev/sdb

Confirm by typing YES (all caps)
```

AES-XTS with 512-bit keys means AES-256 for both the data key and the tweak key. `argon2id` as the PBKDF (Password-Based Key Derivation Function) makes brute-forcing your passphrase expensive.

Step 3: Open the Encrypted Container

```bash
Open the LUKS container and map it to /dev/mapper/mydata
sudo cryptsetup open /dev/sdb mydata

You will be prompted for your passphrase
The device is now available at /dev/mapper/mydata
```

Step 4: Create a Filesystem on the Encrypted Device

```bash
Format with ext4
sudo mkfs.ext4 -L "encrypted-data" /dev/mapper/mydata

Or with XFS for better performance on large files
sudo mkfs.xfs -L "encrypted-data" /dev/mapper/mydata

Create a mount point and mount
sudo mkdir -p /mnt/encrypted
sudo mount /dev/mapper/mydata /mnt/encrypted

Verify
df -h /mnt/encrypted
```

Step 5: Use the Encrypted Partition

```bash
Create files as normal
sudo cp -r /home/user/documents /mnt/encrypted/

Change ownership if you want non-root access
sudo chown -R user:user /mnt/encrypted/
```

Step 6: Unmount and Close

```bash
Unmount the filesystem
sudo umount /mnt/encrypted

Close the LUKS container (re-encrypts key material and removes mapping)
sudo cryptsetup close mydata

Verify it's closed
ls /dev/mapper/
```

After closing, the data on `/dev/sdb` is completely inaccessible without the passphrase.

Step 7: Add a Keyfile (Second Key Slot)

LUKS supports multiple key slots. A keyfile lets you unlock the partition without typing a passphrase. useful for automated unlocking:

```bash
Generate a strong keyfile (512 bytes of random data)
sudo dd if=/dev/urandom of=/root/luks-keyfile bs=512 count=1
sudo chmod 400 /root/luks-keyfile

Add the keyfile to slot 1 (slot 0 has your passphrase)
sudo cryptsetup luksAddKey /dev/sdb /root/luks-keyfile

You will be prompted for your existing passphrase to authorize the addition
```

Now you can open the device with the keyfile instead of the passphrase:
```bash
sudo cryptsetup open --key-file /root/luks-keyfile /dev/sdb mydata
```

Step 8: Automatic Mounting with /etc/crypttab and /etc/fstab

To mount automatically at boot (on a server where the keyfile is on the root disk):

Add to `/etc/crypttab`:
```
name          device      keyfile         options
mydata          /dev/sdb    /root/luks-keyfile  luks
```

Add to `/etc/fstab`:
```
/dev/mapper/mydata  /mnt/encrypted  ext4  defaults,nofail  0  2
```

For removable devices, use `UUID` instead of device path (more reliable):
```bash
Get the UUID of the LUKS container (not the filesystem)
sudo cryptsetup luksDump /dev/sdb | grep UUID

/etc/crypttab
mydata  UUID=a1b2c3d4-...  /root/luks-keyfile  luks
```

Step 9: Inspect LUKS Metadata

```bash
Show key slots and LUKS header info
sudo cryptsetup luksDump /dev/sdb

Check which slot a passphrase uses
sudo cryptsetup --verbose open --test-passphrase /dev/sdb

Remove a key slot (e.g., remove slot 0 passphrase after adding keyfile)
Make sure you have another way in first
sudo cryptsetup luksKillSlot /dev/sdb 0
```

Step 10: Encrypt a Loop File (No Partition Needed)

You can encrypt a file rather than a whole partition. useful for creating portable encrypted containers:

```bash
Create a 5GB container file
dd if=/dev/zero of=/home/user/secure.img bs=1M count=5120

Format it as LUKS
sudo cryptsetup luksFormat /home/user/secure.img

Open it
sudo cryptsetup open /home/user/secure.img securedata

Format and mount
sudo mkfs.ext4 /dev/mapper/securedata
sudo mount /dev/mapper/securedata /mnt/secure

Close when done
sudo umount /mnt/secure && sudo cryptsetup close securedata
```

Backup the LUKS Header

If the LUKS header at the start of the disk is corrupted, the entire partition becomes unrecoverable. Back it up:

```bash
Backup header (does NOT expose key material. safe to store offsite)
sudo cryptsetup luksHeaderBackup /dev/sdb \
  --header-backup-file /root/luks-header-backup.bin

Restore if needed
sudo cryptsetup luksHeaderRestore /dev/sdb \
  --header-backup-file /root/luks-header-backup.bin
```

Store this backup in a separate encrypted location from the device itself.

Related Reading

- [How to Use BorgBackup for Encrypted Backups](/how-to-use-borgbackup-for-encrypted-backups/)
- [Secure Boot Chain Verification on Linux](/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [How to Use AIDE for File Integrity Checking](/how-to-use-aide-for-file-integrity-checking/)

- [How To Create Encrypted Mailing List For Private Group](/how-to-create-encrypted-mailing-list-for-private-group-commu/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

Related Articles

- [How to Create an Encrypted Container with LUKS](/luks-encrypted-container-guide/)
- [LUKS Full Disk Encryption on Linux](/luks-full-disk-encryption-linux-guide/)
- [How to Encrypt an USB Drive with VeraCrypt](/encrypt-usb-drive-veracrypt-guide/)
- [How to Detect Keyloggers on Your System](/how-to-detect-keyloggers-on-your-system/)
- [Best Encrypted Messaging App 2026](/best-encrypted-messaging-app-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
