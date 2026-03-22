---
layout: default
title: "Secure Boot Chain Verification on Linux"
description: "Verify the integrity of your Linux boot chain from UEFI firmware through bootloader and kernel using Measured Boot, TPM PCRs, and attestation"
date: 2026-03-22
author: theluckystrike
permalink: /secure-boot-chain-verification-linux/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Secure Boot prevents unauthorized code from running at startup, but simply having Secure Boot enabled does not tell you whether your boot chain has been compromised. Measured Boot and TPM attestation give you cryptographic proof that each stage — firmware, bootloader, kernel — is exactly what you expect.

## Understanding the Boot Chain

```
UEFI Firmware
    └─ Bootloader (GRUB / systemd-boot)
            └─ Linux Kernel + initrd
                    └─ init / systemd
                            └─ Userspace
```

Each stage can be measured and its hash recorded in the TPM's PCRs (Platform Configuration Registers). A change to any stage produces a different PCR value.

## Step 1: Check Secure Boot Status

```bash
mokutil --sb-state
# SecureBoot enabled

bootctl status 2>/dev/null | grep "Secure Boot"
# Secure Boot: enabled (user)

# Check which keys are enrolled
mokutil --list-enrolled | grep Subject
```

## Step 2: Check TPM Status

```bash
ls /dev/tpm*
# /dev/tpm0  /dev/tpmrm0

cat /sys/class/tpm/tpm0/tpm_version_major
# 2

sudo apt install tpm2-tools
tpm2_getcap properties-fixed | grep -A2 TPM_PT_FIRMWARE
```

## Step 3: Read Current PCR Values

```bash
# Read all PCR values
tpm2_pcrread

# Common PCR assignments (SHA-256):
# PCR 0: UEFI firmware code
# PCR 4: Bootloader
# PCR 7: Secure Boot state and certificates
# PCR 11: systemd-boot entries

tpm2_pcrread sha256:0,4,7,11
```

## Step 4: Record a Known-Good Baseline

After a fresh trusted installation:

```bash
tpm2_pcrread sha256:0,1,2,3,4,5,6,7 > /root/pcr-baseline-$(date +%Y%m%d).txt

# Store offline. On subsequent boots, compare:
tpm2_pcrread sha256:0,1,2,3,4,5,6,7 > /tmp/pcr-current.txt
diff /root/pcr-baseline-*.txt /tmp/pcr-current.txt
```

Any difference indicates a change to the measured component.

## Step 5: Check Bootloader Signatures

```bash
# GRUB
sudo apt install sbsigntool
sbverify --list /boot/efi/EFI/ubuntu/grubx64.efi

# systemd-boot
bootctl status

# Verify kernel signature
pesign --show-signature --in /boot/vmlinuz-$(uname -r)
```

## Step 6: Kernel Module Signature Verification

```bash
# Check module signing enforcement
grep -r "CONFIG_MODULE_SIG" /boot/config-$(uname -r) 2>/dev/null | \
  grep -E "FORCE|REQUIRE"

# List modules without valid signatures
for mod in $(lsmod | awk 'NR>1 {print $1}'); do
    sig=$(modinfo $mod 2>/dev/null | grep "sig_key")
    if [ -z "$sig" ]; then
        echo "UNSIGNED: $mod"
    fi
done
```

## Step 7: TPM-Based Disk Encryption

Bind LUKS disk encryption to TPM PCR values — the disk only auto-unlocks if the boot chain matches:

```bash
# Enroll TPM bound to PCRs 0, 4, 7
sudo systemd-cryptenroll \
  --tpm2-device=auto \
  --tpm2-pcrs=0+4+7 \
  /dev/sda3
```

If any measured component changes (GRUB update, firmware update), the PCR values change and the TPM refuses to release the key. Re-enroll after deliberate updates:

```bash
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sda3
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+4+7 /dev/sda3
```

## Step 8: Remote Attestation

Remote attestation lets a server verify that a client's boot chain is trustworthy before granting access:

```bash
# Generate an attestation quote
tpm2_createprimary -C e -g sha256 -G ecc -c primary.ctx
tpm2_createak -C primary.ctx -c ak.ctx -u ak.pub -n ak.name
tpm2_quote \
  --key-context ak.ctx \
  --pcr-list sha256:0,4,7 \
  --message nonce.bin \
  --signature quote.sig \
  --pcrs quote.pcrs
```

The attestation quote can be sent to a remote server that verifies PCR values match the expected baseline before issuing a VPN certificate or access token.

## Related Articles

- [Secure Boot and TPM Explained for Linux](/privacy-tools-guide/secure-boot-tpm-linux-explained/)
- [Linux Secure Boot Setup with Custom Keys for Preventing](/privacy-tools-guide/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [LUKS Full Disk Encryption on Linux](/privacy-tools-guide/luks-full-disk-encryption-linux-guide/)
- [How to Secure Your Home Router Firmware](/privacy-tools-guide/home-router-firmware-security-guide)
- [How To Set Up Secureboot Plus Encryption On Fedora Linux](/privacy-tools-guide/how-to-set-up-secureboot-plus-encryption-on-fedora-linux-for/)
- [AI Tools for Resolving SSL Certificate Chain Verification](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-resolving-ssl-certificate-chain-verification-er/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
