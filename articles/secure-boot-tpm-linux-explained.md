---
layout: default
title: "Secure Boot and TPM Explained for Linux"
description: "How Secure Boot and TPM 2.0 work on Linux, what threats they address, how to enroll custom keys, and how to use TPM for disk unlock"
date: 2026-03-22
author: theluckystrike
permalink: /secure-boot-tpm-linux-explained/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Secure Boot and TPM 2.0 are often misunderstood as Windows-only features or corporate lock-in tools. On Linux, they can form a real chain of trust — ensuring that only your signed bootloader and kernel load, and that disk unlock happens automatically only when the system hasn't been tampered with.
---

## Table of Contents

- [What Secure Boot Does](#what-secure-boot-does)
- [Enrolling Custom Keys (Full Control)](#enrolling-custom-keys-full-control)
- [What the TPM Does](#what-the-tpm-does)
- [TPM-Based Disk Auto-Unlock (systemd-cryptenroll)](#tpm-based-disk-auto-unlock-systemd-cryptenroll)
- [Measuring Boot with TPM Event Log](#measuring-boot-with-tpm-event-log)
- [sbctl: Easier Custom Key Management](#sbctl-easier-custom-key-management)
- [Checking Secure Boot Status](#checking-secure-boot-status)
- [When Secure Boot Breaks Things](#when-secure-boot-breaks-things)
- [Related Reading](#related-reading)

## What Secure Boot Does

Secure Boot is an UEFI firmware feature that checks the cryptographic signature of every bootloader and kernel before executing it. If the signature doesn't match a trusted key, the firmware refuses to boot.

**What it protects against:**
- Bootkit malware replacing your bootloader
- Someone swapping in a malicious OS from an USB drive
- Cold boot attacks that install persistent rootkits in the boot chain

**What it does NOT protect against:**
- Malware running inside your OS
- Physical access attacks after boot
- Weak disk encryption (evil maid with long access time)

### Default Secure Boot Keys

Out of the box, most systems ship with Microsoft's signing keys. These allow Microsoft-signed bootloaders (including shim, used by Ubuntu/Fedora) but not arbitrary kernels.

```bash
# List current Secure Boot keys
mokutil --list-enrolled
# Or via efitools
efi-readvar -v PK   # Platform Key
efi-readvar -v KEK  # Key Exchange Key
efi-readvar -v db   # Allowed signatures
efi-readvar -v dbx  # Forbidden signatures
```

---

## Enrolling Custom Keys (Full Control)

Custom key enrollment lets you remove Microsoft's keys and only trust your own signed bootloader and kernel. This gives you complete control but requires signing everything yourself.

```bash
# Install required tools
sudo apt install efitools sbsigntool openssl

# Generate your own Platform Key, KEK, and db key
openssl req -newkey rsa:4096 -keyout PK.key -new -x509 -sha256 \
  -days 3650 -subj "/CN=Platform Key/" -out PK.crt

openssl req -newkey rsa:4096 -keyout KEK.key -new -x509 -sha256 \
  -days 3650 -subj "/CN=Key Exchange Key/" -out KEK.crt

openssl req -newkey rsa:4096 -keyout db.key -new -x509 -sha256 \
  -days 3650 -subj "/CN=Signature Database Key/" -out db.crt

# Convert to EFI signature list format
cert-to-efi-sig-list -g "$(uuidgen)" PK.crt PK.esl
cert-to-efi-sig-list -g "$(uuidgen)" KEK.crt KEK.esl
cert-to-efi-sig-list -g "$(uuidgen)" db.crt db.esl

# Sign the signature lists
sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth
sign-efi-sig-list -k PK.key -c PK.crt KEK KEK.esl KEK.auth
sign-efi-sig-list -k KEK.key -c KEK.crt db db.esl db.auth
```

```bash
# Enter Setup Mode in UEFI firmware (clears existing keys)
# Then enroll your keys:
efi-updatevar -f PK.auth PK
efi-updatevar -f KEK.auth KEK
efi-updatevar -f db.auth db

# Sign your kernel and bootloader
sbsign --key db.key --cert db.crt --output /boot/vmlinuz-signed /boot/vmlinuz-current
sbsign --key db.key --cert db.crt --output /boot/efi/EFI/BOOT/BOOTX64.efi.signed \
  /boot/efi/EFI/BOOT/BOOTX64.efi
```

---

## What the TPM Does

The TPM (Trusted Platform Module) is a dedicated security chip on your motherboard. It provides:

- **Measured boot**: Records SHA-256 hashes of each boot component into PCR (Platform Configuration Register) banks
- **Sealed secrets**: Encrypts data so it can only be decrypted when specific PCRs match — i.e., when the system boots in a known-good state
- **Key storage**: Stores cryptographic keys that never leave the chip

```bash
# Check TPM presence and version
cat /sys/class/tpm/tpm0/tpm_version_major  # Should show "2"

# List PCR banks
sudo tpm2_pcrread sha256
```

### PCR Banks That Matter

| PCR | Contains |
|-----|----------|
| 0 | UEFI firmware |
| 4 | Bootloader |
| 7 | Secure Boot state (keys enrolled) |
| 8 | Kernel command line |
| 9 | initrd |

---

## TPM-Based Disk Auto-Unlock (systemd-cryptenroll)

The most practical use of TPM on Linux: automatically decrypt your LUKS disk at boot without typing a passphrase, but only when the system hasn't been tampered with (PCRs match).

```bash
# Install systemd (version 248+ required)
systemd --version

# Enroll TPM into your LUKS volume
# -d specifies the LUKS device, --tpm2-device=auto uses the TPM
sudo systemd-cryptenroll \
  --tpm2-device=auto \
  --tpm2-pcrs=0+4+7 \
  /dev/nvme0n1p3

# PCRs 0+4+7 = firmware + bootloader + Secure Boot state
# If any of these change (tampered firmware, different bootloader),
# auto-unlock fails and you're prompted for the passphrase
```

**Update /etc/crypttab** to use TPM:

```bash
# Edit /etc/crypttab — add tpm2-device=auto option
# Example line:
# luks-UUID /dev/nvme0n1p3 none luks,tpm2-device=auto

sudo sed -i 's/\(luks-[^ ]*\) .* none luks/\1 none none luks,tpm2-device=auto/' /etc/crypttab
sudo update-initramfs -u
```

---

## Measuring Boot with TPM Event Log

```bash
# Read the TPM event log to see what was measured
sudo tpm2_eventlog /sys/kernel/security/tpm0/binary_bios_measurements

# Or use the parsed output
sudo tpm2_pcrread sha256:0,4,7,8,9

# Compare against a known-good baseline to detect tampering
# Save baseline after fresh install:
sudo tpm2_pcrread sha256 > /root/tpm-baseline-$(date +%Y%m%d).txt

# Later comparison:
sudo tpm2_pcrread sha256 | diff /root/tpm-baseline-YYYYMMDD.txt -
```

---

## sbctl: Easier Custom Key Management

`sbctl` is a modern tool that simplifies custom Secure Boot key enrollment:

```bash
# Install sbctl
sudo pacman -S sbctl              # Arch
sudo apt install sbctl            # Debian (may need backports)

# Create new key set
sudo sbctl create-keys

# Enroll keys (keeps Microsoft keys for compatibility, or --microsoft to skip)
sudo sbctl enroll-keys --microsoft
# Without --microsoft: only your keys trusted

# Sign bootloader and kernel
sudo sbctl sign /boot/vmlinuz-linux
sudo sbctl sign /boot/efi/EFI/BOOT/BOOTX64.efi
sudo sbctl sign /boot/efi/EFI/systemd/systemd-bootx64.efi

# Verify signed binaries
sudo sbctl verify

# List all files that need signing
sudo sbctl list-files
```

After a kernel update, re-sign the new kernel:

```bash
# sbctl can track files and auto-sign on pacman/apt hook
sudo sbctl sign --save /boot/vmlinuz-linux
# The --save flag registers it for automatic re-signing
```

---

## Checking Secure Boot Status

```bash
# Quick status check
mokutil --sb-state
# "SecureBoot enabled" = active
# "SecureBoot disabled" = not enforcing

# More detail
bootctl status | grep "Secure Boot"

# Check if UEFI is in setup mode (accepting new keys without auth)
efi-readvar -v SetupMode
```

---

## When Secure Boot Breaks Things

Some kernel modules (particularly out-of-tree drivers: Nvidia, VirtualBox, ZFS) require signing before they'll load with Secure Boot enabled.

```bash
# Sign a module with your db key
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file \
  sha256 \
  /path/to/db.key \
  /path/to/db.crt \
  /lib/modules/$(uname -r)/updates/nvidia.ko

# Alternatively: enroll a MOK (Machine Owner Key) for module signing
sudo mokutil --import db.crt
# Reboot → MOK enrollment screen appears → confirm
```

---

## Related Reading

- [Linux Secure Boot Setup with Custom Keys](/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [LUKS Full Disk Encryption Linux Guide](/luks-full-disk-encryption-linux-guide/)
- [How to Create an Encrypted Container with LUKS](/luks-encrypted-container-guide/)
- [Android Privacy Indicators: Camera and Mic Access Explained](/android-privacy-indicators-camera-mic-explained/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

## Related Articles

- [Linux Secure Boot Setup with Custom Keys for Preventing](/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [Secure Document Collaboration Alternatives to Google](/secure-document-collaboration-alternatives-to-google-docs-wi/)
- [Linux Kernel Hardening with sysctl](/linux-kernel-hardening-sysctl-guide)
- [Set Up Secure Communication For Labor Strike: Practical](/how-to-set-up-secure-communication-for-labor-strike-organizi/)
- [LUKS Full Disk Encryption on Linux](/luks-full-disk-encryption-linux-guide/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.
{% endraw %}
