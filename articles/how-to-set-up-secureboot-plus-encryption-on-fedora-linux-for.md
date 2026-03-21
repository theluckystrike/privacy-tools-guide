---
layout: default
title: "How To Set Up Secureboot Plus Encryption On Fedora Linux"
description: "Combine SecureBoot with LUKS full disk encryption on Fedora to create multi-layer security protecting against bootkits and physical theft. SecureBoot verifies"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-secureboot-plus-encryption-on-fedora-linux-for/
categories: [guides]
tags: [privacy-tools-guide, tools, encryption]
reviewed: true
score: 9
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

### SecureBoot vs. Encryption: Complementary Protections

SecureBoot and LUKS encryption address different threat vectors. SecureBoot prevents an attacker with brief physical access from swapping in a malicious bootloader or kernel—the so-called "evil maid" attack scenario. LUKS encryption prevents an attacker who steals your drive from reading your data offline. Neither layer alone is sufficient: a stolen drive bypasses SecureBoot entirely, while a replaced bootloader can silently capture your LUKS passphrase before encryption even initializes.

Together, they form a chain of trust: the firmware trusts only signed boot components, the signed bootloader unlocks encrypted storage, and your data never reaches an unverified system.

### TPM2 and the Chain of Trust

TPM2 (Trusted Platform Module version 2) strengthens this model by binding decryption keys to measured boot state. Platform Configuration Registers (PCRs) record cryptographic hashes of each boot stage. If any component is modified—firmware, bootloader, or kernel—PCR values change and the TPM refuses to release the decryption key, keeping the drive locked against unauthorized boot paths.

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

For production systems, use a randomly generated passphrase from a password manager and store it in a hardware security key or offline vault. Splitting a long passphrase across two physical locations—a safe deposit box and a sealed envelope with a trusted contact—reduces single-point-of-failure risk.

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

The system will prompt for a one-time password during enrollment. After reboot, you will need to confirm key enrollment through the MOK (Machine Owner Key) manager interface. This blue management screen appears once and disappears after a timeout—if you miss it, power off and back on to see it again.

### Custom Key Enrollment for Maximum Control

Fedora ships with its own signing keys, which means Fedora's infrastructure is part of your trust chain. For environments requiring independent key control, generate your own Platform Key (PK), Key Exchange Key (KEK), and Database key (DB):

```bash
# Generate custom keys (4096-bit RSA for longevity)
openssl req -new -x509 -newkey rsa:4096 -keyout PK.key -out PK.crt -days 3650 \
  -nodes -subj "/CN=Custom Platform Key/"
openssl req -new -x509 -newkey rsa:4096 -keyout KEK.key -out KEK.crt -days 3650 \
  -nodes -subj "/CN=Custom KEK/"
openssl req -new -x509 -newkey rsa:4096 -keyout DB.key -out DB.crt -days 3650 \
  -nodes -subj "/CN=Custom Signing Key/"

# Convert to EFI signature lists
cert-to-efi-sig-list PK.crt PK.esl
cert-to-efi-sig-list KEK.crt KEK.esl
cert-to-efi-sig-list DB.crt DB.esl
```

Enrolling custom keys requires putting your firmware into Setup Mode first, which clears all existing Secure Boot variables. Only do this if you are prepared to sign every boot component yourself—including shim, grub, and the kernel. This approach suits high-security deployments and air-gapped systems.

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

### Automating Kernel Signing with Hooks

Fedora kernel updates are frequent; manual signing after each update is error-prone and easy to forget. Automate signing with a kernel install hook placed in `/etc/kernel/install.d/`:

```bash
#!/bin/bash
# /etc/kernel/install.d/99-sign-kernel.install

COMMAND="$1"
KERNEL_VERSION="$2"
BOOT_DIR_ABS="$3"

if [[ "$COMMAND" == "add" ]]; then
    sbsign --key /etc/secureboot/db.key \
           --cert /etc/secureboot/db.crt \
           --output "${BOOT_DIR_ABS}/vmlinuz" \
           "${BOOT_DIR_ABS}/vmlinuz"
    echo "Kernel ${KERNEL_VERSION} signed for SecureBoot"
fi
```

Make it executable with `chmod +x` and store signing keys in `/etc/secureboot/` with permissions `600`. This hook runs automatically on each `dnf upgrade` that installs a new kernel, keeping your chain of trust intact without manual intervention.

## Step 4: Configuring TPM2-Based Unlocking

TPM2 integration allows automatic decryption without compromising security:

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

### Choosing the Right PCRs

The `--tpm2-pcrs` argument controls which boot measurements must match for automatic unlock. Selecting the wrong PCRs either weakens security or causes constant unlock failures after routine updates. Here is a practical breakdown:

| PCRs | What It Covers | Risk If Excluded |
|------|---------------|------------------|
| 0 | Firmware code | Firmware-level rootkits go undetected |
| 1 | Firmware configuration | BIOS setting changes bypass the TPM check |
| 4 | Bootloader code | Bootloader swap attack succeeds |
| 7 | SecureBoot state and policy | Disabling SecureBoot bypasses the TPM check |
| 8 | Kernel command line | Kernel parameter injection attacks succeed |
| 9 | Initrd and kernel image | Tampered initrd runs before decryption |

For most threat models, PCRs `0+4+7` provides solid protection with minimal disruption from legitimate firmware updates. Adding PCR 8 locks the kernel command line, which can require re-enrollment after any `grubby` change. Adding PCR 1 is appropriate for air-gapped systems where firmware configuration never legitimately changes.

This setup provides multi-factor authentication: the TPM must confirm system integrity, and your passphrase unlocks the key if TPM verification fails.

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

**Restrict kernel memory access:**

Add these kernel parameters to `/etc/default/grub` to close information-leakage channels an attacker could use to extract encryption keys from a running system:

```bash
GRUB_CMDLINE_LINUX="... lockdown=confidentiality module.sig_enforce=1"
```

The `lockdown=confidentiality` parameter prevents userspace processes from reading kernel memory through `/dev/mem`, `/dev/kmem`, and similar interfaces. It also disables loading of unsigned kernel modules. Run `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` after editing to regenerate the bootloader configuration.

## Troubleshooting Common Issues

If you encounter boot failures after enabling SecureBoot:

1. **No boot device detected**: Re-enroll SecureBoot keys through firmware settings
2. **LUKS passphrase not accepted**: Boot from live USB and run `fsck` on encrypted partition
3. **TPM unlock fails**: Ensure TPM2 module loads early in initramfs by adding to `/etc/dracut.conf`:

```bash
add_drivers+="tpm_tis tpm_crb"
```

4. **Kernel signature verification fails**: Re-sign the kernel or rebuild with signed modules
5. **PCR mismatch after firmware update**: Firmware updates legitimately change PCR 0. After any BIOS/UEFI update, re-enroll the TPM2 binding with `sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/nvme0n1p3` followed by a fresh `--tpm2-device=auto` enrollment
6. **MOK enrollment screen not appearing**: The MOK manager has a short timeout. Power cycle and watch immediately after POST to catch the enrollment prompt

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

Test your recovery procedure at least once per year by booting from a live USB and restoring access using the backup passphrase. An untested recovery plan is not a recovery plan—discovering the backup is wrong during an actual incident causes permanent data loss.

## Related Reading

- [Pop Os Vs Fedora Vs Debian For Privacy Focused Linux Desktop](/privacy-tools-guide/pop-os-vs-fedora-vs-debian-for-privacy-focused-linux-desktop/)
- [How to Set Up S/MIME Email Encryption: A Practical Guide](/privacy-tools-guide/how-to-set-up-smime-email-encryption/)
- [Secure Email Forwarding With Encryption How To Set Up Anonad](/privacy-tools-guide/secure-email-forwarding-with-encryption-how-to-set-up-anonad/)
- [C++ Developer Privacy Tools for Detecting Telemetry in.](/privacy-tools-guide/c-plus-plus-developer-privacy-tools-for-detecting-telemetry-in-game-engine-sdks-2026/)
- [Use Email Subaddressing Plus Addressing For Tracking Which](/privacy-tools-guide/how-to-use-email-subaddressing-plus-addressing-for-tracking-which-services-leak-your-address/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
