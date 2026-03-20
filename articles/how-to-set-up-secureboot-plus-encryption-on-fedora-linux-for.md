---
layout: default
title: "How To Set Up Secureboot Plus Encryption On Fedora Linux For"
description: "A comprehensive guide to configuring SecureBoot and full disk encryption on Fedora Linux for enhanced security. Step-by-step instructions for."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-secureboot-plus-encryption-on-fedora-linux-for/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

Combine SecureBoot with LUKS full disk encryption on Fedora to create multi-layer security protecting against bootkits and physical theft. SecureBoot verifies bootloader signatures before execution, while TPM2 integration allows automatic decryption without compromising security. This guide covers key enrollment, kernel signing, automatic signing via hooks, and recovery planning for production Linux systems.

## Prerequisites

Before proceeding, ensure you have:

- A Fedora Linux system (Fedora 40 or later recommended)
- UEFI firmware with SecureBoot capability
- At least 50GB free disk space for a fresh installation
- A backup of important data

## Understanding the Security Architecture

SecureBoot operates at the firmware level, verifying cryptographic signatures of boot components before execution. When combined with LUKS (Linux Unified Key Setup) encryption, you create a multi-layered security model:

- **Pre-boot authentication**: SecureBoot validates the bootloader
- **Bootloader verification**: The bootloader verifies the kernel signature
- **Encrypted storage**: All data remains inaccessible without proper key material

This approach protects against bootkits, ransomware targeting the boot process, and physical theft of storage devices.

## Step 1: Preparing Fedora Installation with Encryption

During Fedora installation, select the "Advanced Custom" partitioning option to configure encryption properly:

```bash
# If using the Anaconda installer interactively:
# 1. Choose "Custom" partitioning
# 2. Create /boot as ext4 (unencrypted for bootloader)
# 3. Create root (/) as LUKS encrypted partition
# 4. Enable "Encrypt my data" checkbox
```

For command-line automated installations, create a Kickstart file:

```kickstart
# encrypt-fedora.ks
part /boot --fstype=ext4 --size=512
part pv.01 --grow
volgroup vg_root pv.01
logvol / --fstype=xfs --size=10240 --name=lv_root --vgname=vg_root --encrypted --passphrase=YOUR_SECURE_PASSPHRASE
bootloader --location=mbr --encrypted
```

Replace `YOUR_SECURE_PASSPHRASE` with a strong passphrase meeting these criteria:

- Minimum 20 characters
- Mix of uppercase, lowercase, numbers, and symbols
- No dictionary words or personal information

## Step 2: Enrolling SecureBoot Keys

After installation, enroll the Fedora SecureBoot keys through mokutil:

```bash
# Check current SecureBoot status
mokutil --sb-state

# Enroll the Fedora shim and bootloader keys
sudo mokutil --enroll-key /usr/share/shim-signed/shimx64.efi.signed.key

# Alternatively, enroll from the default Fedora keys
sudo mokutil --import /usr/share/secureboot/keys/fedora/DB.key
```

The system will prompt for a one-time password during enrollment. After reboot, you'll need to confirm key enrollment through the MOK (Machine Owner Key) manager interface.

## Step 3: Signing Custom Kernels and Bootloaders

For developers who compile custom kernels, signing becomes essential:

```bash
# Generate a signing key
openssl req -new -x509 -newkey rsa:2048 -keyout kernel-signing.key -out kernel-signing.pem -days 3650 -nodes -subj "/CN=Kernel Signing/"

# Sign the kernel
/usr/bin/sign-file sha256 kernel-signing.key kernel-signing.pem /boot/vmlinuz-$(uname -r)

# Verify the signature
tail /boot/vmlinuz-$(uname -r) | grep "PE signature"
```

Configure the system to require signed modules:

```bash
# Add to /etc/dracut.conf.d/signed-modules.conf
add_drivers+="tpm"

# Rebuild initramfs with module signing
sudo dracut --force --regenerate-all
```

## Step 4: Configuring TPM2-Based Unlocking

TPM2 (Trusted Platform Module) integration allows automatic decryption without compromising security:

```bash
# Install TPM2 tools
sudo dnf install tpm2-tools

# Create a TPM-backed key file
sudo tpm2_createprimary -C e -g sha256 -G rsa -c primary.ctx
sudo tpm2_create -C primary.ctx -u sealed.key -r unsealed.key

# Add the sealed key to LUKS
sudo cryptsetup luksAddKey /dev/nvme0n1p3 sealed.key
```

Configure `systemd-cryptenroll` for automatic unlock:

```bash
# Enroll TPM2 with LUKS
sudo systemd-cryptenroll /dev/nvme0n1p3 --tpm2-device=auto --tpm2-pcrs=0+1+2+3+4+5+6+7+8

# Verify enrollment
sudo systemd-cryptenroll /dev/nvme0n1p3 --tpm2-device=auto
```

This setup requires both the TPM measurement and your passphrase for decryption, providing multi-factor authentication for your encrypted drive.

## Step 5: Verifying Your Security Configuration

After configuration, verify each component:

```bash
# Confirm SecureBoot is enabled
mokutil --sb-state
# Expected output: SecureBoot enabled

# Check LUKS encryption status
sudo cryptsetup luksDump /dev/nvme0n1p3 | grep -i "Encrypted"
# Should show "LUKS2" or "LUKS1"

# Verify kernel module signature requirements
cat /proc/cmdline | grep -i "module.sig_enforce"
# Should show module.sig_enforce=1

# Test TPM unlock (requires manual reboot)
sudo systemd-cryptsetup attach data /dev/nvme0n1p3 "tpm2-device=auto"
```

## Step 6: Implementing Additional Hardening

Strengthen your configuration with these optional measures:

**Disable kernel dump to encrypted swap:**

```bash
sudo systemctl mask kdump.service
sudo systemctl mask dm-crypt-generator.service
```

**Enable SecureBoot in FIPS mode:**

```bash
sudo fips-mode-setup --enable --boot-location=/boot
sudo reboot
```

**Configure automatic locking on suspend:**

Edit `/etc/systemd/logind.conf`:

```ini
HandleSuspendKey=lock
HandleLidSwitch=lock
IdleAction=lock
IdleActionSec=15min
```

## Troubleshooting Common Issues

If you encounter boot failures after enabling SecureBoot:

1. **No boot device detected**: Re-enroll SecureBoot keys through firmware settings
2. **LUKS passphrase not accepted**: Boot from live USB and run `fsck` on encrypted partition
3. **TPM unlock fails**: Ensure TPM2 module loads early in initramfs by adding to `/etc/mkinitcpio.conf`:

```bash
MODULES=(tpm_tis tpm2_software)
```

4. **Kernel signature verification fails**: Re-sign the kernel or rebuild with signed modules

## Maintenance and Recovery

Regular maintenance ensures continued security:

```bash
# Rotate encryption keys annually
sudo cryptsetup luksChangeKey /dev/nvme0n1p3

# Update SecureBoot keys before expiration
sudo mokutil --export
# Import new keys and remove expired ones

# Verify boot integrity on each startup
sudo bootctl verify
```

Store your LUKS recovery passphrase in a secure location separate from your device. Consider using a hardware security key as an additional authentication factor.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
