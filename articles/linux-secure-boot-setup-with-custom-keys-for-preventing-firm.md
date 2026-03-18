---
layout: default
title: "Linux Secure Boot Setup with Custom Keys for Preventing Firmware Level Tampering"
description: "Learn how to set up Secure Boot on Linux with custom keys to protect against firmware-level attacks and ensure boot integrity."
date: 2026-03-16
author: theluckystrike
permalink: /linux-secure-boot-setup-with-custom-keys-for-preventing-firm/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
---

{% raw %}
# Linux Secure Boot Setup with Custom Keys for Preventing Firmware Level Tampering

Modern systems face increasingly sophisticated threats at the firmware level. Attackers who compromise the boot process can bypass operating system security controls entirely, making detection extremely difficult. Secure Boot provides a defense mechanism by verifying cryptographic signatures before executing bootloader code. However, most systems ship with Microsoft keys pre-installed, which limits your control over what gets trusted during the boot process.

Setting up Secure Boot with your own keys gives you complete authority over the trust chain. This approach ensures that only bootloaders and kernels you explicitly sign can execute, preventing malicious firmware modifications or unauthorized operating system boots.

## Prerequisites

Before beginning, gather the following:

- A Linux system with UEFI firmware supporting Secure Boot
- `sbsigntool` and `efitools` packages
- Root or sudo access
- Understanding of your system's firmware menu key combinations

Install the required tools on your current distribution:

```bash
# Debian/Ubuntu
sudo apt install sbsigntool efitools

# Fedora/RHEL
sudo dnf install sbsigntool efitools

# Arch Linux
sudo pacman -S sbsigntool efitools
```

## Generating Your Custom Keys

Secure Boot uses RSA key pairs—public keys get enrolled in your firmware, while private keys sign your boot components. Generate a 2048-bit key set using OpenSSL:

```bash
# Create a directory for keys
mkdir -p ~/secureboot/keys
cd ~/secureboot/keys

# Generate the Platform Key (PK)
openssl genrsa -out PK.key 2048
openssl req -new -x509 -key PK.key -out PK.crt -days 3650

# Generate the Key Exchange Key (KEK)
openssl genrsa -out KEK.key 2048
openssl req -new -x509 -key KEK.key -out KEK.crt -days 3650

# Generate the Signature Database key (db)
openssl genrsa -out db.key 2048
openssl req -new -x509 -key db.key -out db.crt -days 3650
```

The **Platform Key** serves as the root of trust—only this key can modify other keys in Secure Boot. The **Key Exchange Key** controls what keys can sign executables in the permitted database. The **db** key signs your actual boot components.

Protect your private keys with appropriate filesystem permissions:

```bash
chmod 600 *.key
chmod 644 *.crt
```

## Enrolling Keys in UEFI Firmware

This step requires accessing your system's UEFI setup utility. The exact key varies by manufacturer—common options include F2, F10, Delete, or Esc. Look for the Secure Boot configuration section.

The enrollment process typically involves converting certificates to EFI format:

```bash
# Convert certificates to EFI signature list format
cert-to-efi-sig-list -g $(uuidgen) PK.crt PK.esl
cert-to-efi-sig-list -g $(uuidgen) KEK.crt KEK.esl
cert-to-efi-sig-list -g $(uuidgen) db.crt db.esl

# Sign the KEK and db entries with the PK
sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.signed
sign-efi-sig-list -k PK.key -c PK.crt KEK KEK.esl KEK.signed
sign-efi-sig-list -k PK.key -c PK.crt db db.esl db.signed
```

The exact enrollment method depends on your firmware. Some systems provide a "Key Management" interface where you can enroll `.esl` or `.cer` files directly. Others require using `efi-updatevar` from a bootable environment:

```bash
# From a live USB with Secure Boot disabled
sudo efi-updatevar -f db.esl db
sudo efi-updatevar -f KEK.esl KEK
sudo efi-updatevar -f PK.esl PK
```

For systems with limited firmware options, tools like `keytool` from the efitools package provide a graphical interface for key enrollment.

## Signing Boot Components

With keys enrolled, you must sign every component your system boots. This includes the bootloader, kernel, and any intermediate loaders.

Sign your GRUB bootloader:

```bash
# Sign the GRUB binary
sbsign --key ~/secureboot/keys/db.key --cert ~/secureboot/keys/db.crt \
    --output /boot/efi/EFI/*/grubx64.efi /boot/efi/EFI/*/grubx64.efi
```

For systems using systemd-boot, sign the bootloader similarly:

```bash
sbsign --key ~/secureboot/keys/db.key --cert ~/secureboot/keys/db.crt \
    --output /boot/efi/EFI/systemd/systemd-bootx64.efi \
    /boot/efi/EFI/systemd/systemd-bootx64.efi
```

If you use a unified kernel image, sign the entire kernel bundle:

```bash
sbsign --key ~/secureboot/keys/db.key --cert ~/secureboot/keys/db.crt \
    --output /boot/vmlinuz-$(uname -r).signed \
    /boot/vmlinuz-$(uname -r)
```

## Configuring Automatic Signing

Rebuilding and manually signing after every kernel update quickly becomes tedious. Automate this process with a kernel hook:

```bash
sudo tee /etc/kernel/postinst.d/secure-boot-sign > /dev/null << 'EOF'
#!/bin/bash
KERNEL_VERSION="$1"
BOOT_DIR="$2"

sbsign --key /home/*/secureboot/keys/db.key \
       --cert /home/*/secureboot/keys/db.crt \
       --output "${BOOT_DIR}/vmlinuz-${KERNEL_VERSION}.signed" \
       "${BOOT_DIR}/vmlinuz-${KERNEL_VERSION}" 2>/dev/null || true
EOF

sudo chmod +x /etc/kernel/postinst.d/secure-boot-sign
```

Adjust paths according to your setup. This hook triggers after kernel installations, automatically signing new kernels.

## Verification and Testing

After completing setup, verify your configuration:

```bash
# Check if Secure Boot is enabled
mokutil --sb-state

# List enrolled keys
mokutil --list-enrolled

# Verify a signed binary
sbverify --cert ~/secureboot/keys/db.crt /boot/efi/EFI/*/grubx64.efi
```

Successful verification produces output indicating the signature is valid.

Test the setup by rebooting into your firmware settings and examining the Secure Boot status. Your custom keys should appear in the key management interface.

## Recovery Planning

Custom Secure Boot configurations carry a risk—losing access to your keys or making enrollment mistakes can render your system unbootable. Create backup media before proceeding:

```bash
# Export keys to encrypted archive
tar czf secureboot-keys-backup.tar.gz -C ~/secureboot keys/
gpg -o secureboot-keys-backup.tar.gz.gpg --symmetric secureboot-keys-backup.tar.gz
```

Store this backup securely, preferably offline. Additionally, note your firmware's "Clear Secure Boot Keys" option exists as a last-resort recovery method—this returns to factory defaults.

## Conclusion

Implementing custom Secure Boot keys transforms your system's security posture. Rather than relying on manufacturer-trusted certificates, you gain cryptographic assurance that only your signed code executes during boot. While the initial setup requires effort, the protection against firmware-level tampering justifies the investment.

For developers handling sensitive workloads or security-conscious power users, this configuration provides peace of mind knowing your boot chain remains uncompromised. The keys you generate today protect your system tomorrow.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
