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

Secure Boot prevents unauthorized code from running at startup, but simply having Secure Boot enabled does not tell you whether your boot chain has been compromised. Measured Boot and TPM attestation give you cryptographic proof that each stage. firmware, bootloader, kernel. is exactly what you expect.

Understanding the Boot Chain

```
UEFI Firmware
     Bootloader (GRUB / systemd-boot)
             Linux Kernel + initrd
                     init / systemd
                             Userspace
```

Each stage can be measured and its hash recorded in the TPM's PCRs (Platform Configuration Registers). A change to any stage produces a different PCR value.

Step 1 - Check Secure Boot Status

```bash
mokutil --sb-state
SecureBoot enabled

bootctl status 2>/dev/null | grep "Secure Boot"
Secure Boot - enabled (user)

Check which keys are enrolled
mokutil --list-enrolled | grep Subject
```

Step 2 - Check TPM Status

```bash
ls /dev/tpm*
/dev/tpm0  /dev/tpmrm0

cat /sys/class/tpm/tpm0/tpm_version_major
2

sudo apt install tpm2-tools
tpm2_getcap properties-fixed | grep -A2 TPM_PT_FIRMWARE
```

Step 3 - Read Current PCR Values

```bash
Read all PCR values
tpm2_pcrread

Common PCR assignments (SHA-256):
PCR 0 - UEFI firmware code
PCR 4 - Bootloader
PCR 7 - Secure Boot state and certificates
PCR 11 - systemd-boot entries

tpm2_pcrread sha256:0,4,7,11
```

Step 4 - Record a Known-Good Baseline

After a fresh trusted installation:

```bash
tpm2_pcrread sha256:0,1,2,3,4,5,6,7 > /root/pcr-baseline-$(date +%Y%m%d).txt

Store offline. On subsequent boots, compare:
tpm2_pcrread sha256:0,1,2,3,4,5,6,7 > /tmp/pcr-current.txt
diff /root/pcr-baseline-*.txt /tmp/pcr-current.txt
```

Any difference indicates a change to the measured component.

Step 5 - Check Bootloader Signatures

```bash
GRUB
sudo apt install sbsigntool
sbverify --list /boot/efi/EFI/ubuntu/grubx64.efi

systemd-boot
bootctl status

Verify kernel signature
pesign --show-signature --in /boot/vmlinuz-$(uname -r)
```

Step 6 - Kernel Module Signature Verification

```bash
Check module signing enforcement
grep -r "CONFIG_MODULE_SIG" /boot/config-$(uname -r) 2>/dev/null | \
  grep -E "FORCE|REQUIRE"

List modules without valid signatures
for mod in $(lsmod | awk 'NR>1 {print $1}'); do
    sig=$(modinfo $mod 2>/dev/null | grep "sig_key")
    if [ -z "$sig" ]; then
        echo "UNSIGNED: $mod"
    fi
done
```

Step 7 - TPM-Based Disk Encryption

Bind LUKS disk encryption to TPM PCR values. the disk only auto-unlocks if the boot chain matches:

```bash
Enroll TPM bound to PCRs 0, 4, 7
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

Step 8 - Remote Attestation

Remote attestation lets a server verify that a client's boot chain is trustworthy before granting access:

```bash
Generate an attestation quote
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

Automated Boot Chain Verification Script

Monitor your boot chain continuously:

```bash
#!/bin/bash
boot-verify.sh. automated PCR monitoring

BASELINE="/root/pcr-baseline.txt"
CURRENT="/tmp/pcr-current.txt"
ALERT_EMAIL="admin@example.com"

Read current PCR values
tpm2_pcrread sha256:0,1,2,3,4,5,6,7 > "$CURRENT"

Compare to baseline
if ! diff "$BASELINE" "$CURRENT" > /dev/null; then
    echo "WARNING: Boot chain modified!" | mail -s "TPM PCR Mismatch Alert" "$ALERT_EMAIL"
    echo "Baseline:" && cat "$BASELINE"
    echo "Current:" && cat "$CURRENT"
    diff "$BASELINE" "$CURRENT"
    exit 1
fi

echo "Boot chain verified OK at $(date)"
exit 0
```

Run this at startup via cron:
```bash
/etc/cron.d/boot-verify
@reboot root /root/boot-verify.sh
```

UEFI Firmware Validation

Beyond kernel signing, verify the firmware itself has not been modified:

```bash
Check firmware updates applied
fwupdmgr get-devices
fwupdmgr get-history

If using Dell, HP, or Lenovo firmware:
Download the BIOS file and verify its signature
Most OEMs publish GPG keys for BIOS firmware

Dell
gpg --recv-key 0x1234ABCD  # Dell's key
gpg --verify BIOS_update.sig BIOS_update.bin
```

Firmware is rarely compromised but is the hardest component to validate post-deployment. This verification works only if you establish a trusted baseline immediately after OS installation.

TPM Limitations and Assumptions

TPM Measured Boot does NOT protect against:

1. Supply chain attacks: Compromised firmware at manufacturing
2. Evil maid attacks: Physical replacement of boot components
3. Rootkits loaded after boot: Only the boot chain is measured, not runtime code
4. Insider threats: Someone with root access can re-measure and create new baselines

TPM IS effective against:
1. Software-based BIOS/firmware rootkits
2. Unauthorized bootloader modifications
3. Kernel tampering
4. Detecting when your system was compromised (post-incident)

It's an audit tool, not a complete security solution. Use alongside:
- Full disk encryption (LUKS)
- Immutable system binaries (dm-verity)
- Audit logging (auditd)
- File integrity monitoring (aide, tripwire)

Bootloader Hardening Beyond Secureboot

GRUB Password Protection

```bash
Generate a password hash
grub-mkpasswd-pbkdf2

Edit /etc/grub.d/40_custom and add:
set superusers="root"
password_pbkdf2 root grub.pbkdf2.sha512.10000.HASH_HERE

Rebuild GRUB config
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Now GRUB menu is password protected. cannot boot into single-user mode without the password.

Immutable Boot Directory (systemd-protect-system)

On systemd systems, mark the boot directory immutable:

```bash
sudo chattr +i /boot
sudo chattr +i /boot/vmlinuz-*
```

An attacker cannot modify the kernel even with root access (requires special CAP_LINUX_IMMUTABLE capability).

Validating Your Own System Configuration

Create a boot chain report:

```bash
#!/bin/bash
system-security-report.sh

echo "=== SECURE BOOT CONFIGURATION ==="
mokutil --sb-state
bootctl status

echo -e "\n=== TPM STATUS ==="
ls /dev/tpm* 2>/dev/null && echo "TPM present" || echo "TPM NOT present"
tpm2_getcap properties-fixed 2>/dev/null | grep TPM_PT_FIRMWARE

echo -e "\n=== PCR VALUES (SHA256) ==="
tpm2_pcrread sha256:0,4,7,11

echo -e "\n=== BOOTLOADER SIGNING ==="
sbverify --list /boot/efi/EFI/*/grubx64.efi 2>/dev/null

echo -e "\n=== KERNEL MODULE SIGNING ==="
grep -r "CONFIG_MODULE_SIG" /boot/config-$(uname -r) 2>/dev/null

echo -e "\n=== DISK ENCRYPTION STATUS ==="
cryptsetup luksDump /dev/sda3 2>/dev/null | grep "TPM"

echo -e "\n=== AUDIT LOG CONFIG ==="
auditctl -l | head -5
```

Run regularly:
```bash
chmod +x system-security-report.sh
./system-security-report.sh > security-baseline-$(date +%Y%m%d).txt
```

Related Articles

- [Secure Boot and TPM Explained for Linux](/secure-boot-tpm-linux-explained/)
- [Linux Secure Boot Setup with Custom Keys for Preventing](/linux-secure-boot-setup-with-custom-keys-for-preventing-firm/)
- [LUKS Full Disk Encryption on Linux](/luks-full-disk-encryption-linux-guide/)
- [How to Secure Your Home Router Firmware](/home-router-firmware-security-guide)
- [How To Set Up Secureboot Plus Encryption On Fedora Linux](/how-to-set-up-secureboot-plus-encryption-on-fedora-linux-for/)
- [AI Tools for Resolving SSL Certificate Chain Verification](https://bestremotetools.com/ai-tools-for-resolving-ssl-certificate-chain-verification-er/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
