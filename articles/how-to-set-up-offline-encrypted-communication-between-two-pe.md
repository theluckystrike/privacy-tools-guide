---
layout: default
title: "How To Set Up Offline Encrypted Communication Between Two"
description: "When internet connectivity is unavailable or deliberately avoided, establishing secure communication between two parties requires a different approach. Offline"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-offline-encrypted-communication-between-two-pe/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

When internet connectivity is unavailable or deliberately avoided, establishing secure communication between two parties requires a different approach. Offline encrypted communication eliminates network-based attack vectors entirely, making it ideal for high-threat environments, air-gapped systems, or situations where digital footprint minimization is critical. This guide covers practical methods for developers and power users to set up encrypted channels without relying on online infrastructure.

## Understanding the Threat Model

Offline communication still faces security challenges. The primary threats include physical device compromise, key interception during exchange, and forensic analysis of storage media. Before implementing a solution, identify your adversaries—state-level actors, corporate surveillance, or opportunistic attackers—because this determines the appropriate encryption strength and operational security procedures.

End-to-end encryption remains the foundation. Even if an attacker gains physical access to one endpoint, they should not be able to read intercepted messages without the corresponding decryption key. This principle drives all the methods discussed below.

## Method 1: GPG Encryption with Physical Key Exchange

GNU Privacy Guard (GPG) provides a mature, well-audited encryption system that works completely offline. The core idea is simple: generate an asymmetric key pair, exchange public keys physically (via USB drive, SD card, or QR code), then use those keys to encrypt messages or files.

### Step 1: Generate Your Key Pair

Install GPG and generate a new key:

```bash
# Install GPG (macOS)
brew install gnupg

# Generate a new key pair
gpg --full-generate-key
```

Select RSA with 4096 bits for maximum security. Enter your identity information and a strong passphrase. This creates both your private key (never share this) and public key.

### Step 2: Export Your Public Key

```bash
# Export your public key to a file
gpg --armor --export your@email.com > yourname-public.asc

# Export your private key (backup only, never share)
gpg --armor --export-secret-keys your@email.com > backup-private.asc
```

### Step 3: Exchange Keys Physically

Transfer the public key file via USB drive, or convert it to a QR code for visual exchange:

```bash
# Convert public key to QR code (requires qrencode)
qrencode -t ansiutf8 < yourname-public.asc
```

### Step 4: Encrypt Messages

Once you have your correspondent's public key, encrypt messages like this:

```bash
# Encrypt a message file
gpg --encrypt --recipient correspondent@email.com --output message.gpg message.txt

# Decrypt a received message
gpg --decrypt message.gpg
```

GPG also supports symmetric encryption if you prefer a shared password rather than key pairs:

```bash
# AES-256 symmetric encryption
gpg --symmetric --cipher-algo AES256 --output secret.gpg plaintext.txt
```

## Method 2: Signal Protocol with Offline Message Storage

The Signal Protocol provides forward secrecy and deniability, but standard Signal requires network connectivity. However, you can use the underlying library (libsignal) to build offline-capable applications or adapt backup-based workflows.

For truly offline use, consider exporting Signal chats as encrypted backups and transferring them physically:

```bash
# Note: Signal Desktop allows chat export (Settings > Privacy > Advanced)
# Export is encrypted with a backup password
```

For developers, libsignal-java or libsignal-swift can be integrated into custom applications that store encrypted messages locally for physical transfer.

## Method 3: WireGuard for Encrypted Tunnel Over Any Medium

WireGuard creates encrypted tunnels that work over any IP-based transport. While typically used for network tunnels, WireGuard can tunnel data through USB serial connections, Bluetooth, or even acoustic modems between air-gapped systems.

### Setting Up WireGuard Over USB Serial

On both machines, install WireGuard and set up a serial tunnel:

```bash
# Machine A: Create a WireGuard interface
wg genkey | tee privateA | wg pubkey > publicA

# Machine B
wg genkey | tee privateB | wg pubkey > publicB
```

Configure `/etc/wireguard/wg0.conf` on both ends with the other's public key. The endpoint addresses can be RFC 1918 private addresses assigned to the USB serial connection.

While this approach is more complex, it provides full tunnel encryption for any data transferred, not just files or messages.

## Method 4: Custom Encryption with age

The `age` tool (pronounced like the Italian "aghe") is a modern, simple encryption tool designed for file encryption. It supports both password-based and key-based encryption with a focus on simplicity and security.

```bash
# Install age
brew install age

# Generate a key
age-keygen -o age.key

# Encrypt a file
age -p -o encrypted.txt.age plain.txt

# Decrypt
age -d -i age.key encrypted.txt.age
```

For offline exchange, generate keys on both machines, exchange public keys via USB, then encrypt files:

```bash
# Encrypt using recipient's public key
age -r age-recipient-public-key < plain.txt > encrypted.txt.age
```

## Operational Security Considerations

Technical encryption alone does not guarantee security. Consider these operational practices:

**Air-gapped systems**: For maximum security, keep sensitive machines disconnected from all networks. Use a dedicated transfer device (formatted before and after each use) to move data between air-gapped and online systems.

**Key storage**: Store private keys on hardware security modules (HSMs) or encrypted USB drives. Consider using paper backups with QR codes for long-term archival.

**Metadata minimization**: Even encrypted communications reveal metadata—timing, frequency, message length. Use constant-rate padding or mix networks to obscure these patterns.

**Verification**: Always verify key fingerprints through a separate channel. For high-security scenarios, verify in person using government-issued identification.

**Key rotation**: Regularly rotate encryption keys to limit the impact of potential future compromises. GPG supports key expiration and rotation natively.

## Recommended Workflow for Developers

For developers building offline-capable systems, consider this architecture:

1. Generate X25519 key pairs using libsodium or Go's crypto/x25519 package
2. Use XChaCha20-Poly1305 for authenticated encryption of message payloads
3. Store encrypted messages in a simple append-only log format
4. Transfer via USB with filesystem-level encryption (LUKS or FileVault)
5. Implement forward secrecy by rotating session keys periodically

Example Go code for X25519 key exchange:

```go
package main

import (
    "crypto/ed25519"
    "crypto/rand"
    "fmt"
)

func main() {
    // Generate key pair for Alice
    pubKey, privKey, err := ed25519.GenerateKey(rand.Reader)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Alice's public key: %x\n", pubKey)
    fmt.Printf("Alice's private key (keep secret): %x\n", privKey)
}
```

## Using Physical Media for Large Transfers

For scenarios requiring offline transfer of large amounts of data, physical media requires special handling.

### USB-based Transfer Protocol

For large file exchanges between air-gapped systems:

```bash
# On source machine: Create encrypted USB with password
gpg --symmetric --cipher-algo AES256 --output large_archive.gpg large_data.tar

# Transfer USB physically
# Verify USB integrity before trusting content
sha256sum large_data.tar | tee checksum.txt
gpg --decrypt --output - large_archive.gpg | sha256sum

# Compare checksums—if they match, USB wasn't corrupted or tampered
```

### Hard Drive Wiping Before Reuse

After using physical media for sensitive transfers, proper wiping prevents data recovery:

```bash
# NIST-approved multi-pass wipe
shred -vfz -n 7 /path/to/file  # 7-pass overwrite

# Or use dedicated tools for complete drive wipe
sudo zerofree -v /dev/usb-device  # For flash drives
```

Flash drives require special attention because worn cells may not overwrite reliably. Consider destroying critical-use flash drives rather than reusing them.

## Implementing Perfect Forward Secrecy

Offline communication can implement PFS by rotating session keys:

```bash
# Generate a sequence of one-time keys
for i in {1..100}; do
  age-keygen -o key-$i.txt
  cat key-$i.txt
done

# Each message uses the next key in sequence
# Once used, delete the key—it cannot decrypt future messages
```

This means even if an attacker obtains your key file, they can only decrypt messages using that specific key, not all past or future messages.

## Disaster Recovery in Air-Gapped Scenarios

For long-term secure storage of encrypted data:

```bash
# Create redundant backups on different media types
# Optical media (M-Disc): 50+ year lifespan
# Store in multiple geographic locations
# Test recovery procedures annually

# Verify backup integrity
tar tzf encrypted_backup.tar.gz > /dev/null && echo "Backup OK"
```

## When to Use Offline Encryption

Offline encryption is appropriate for:
- State-level threat environments where network monitoring is ubiquitous
- High-value intellectual property or trade secrets
- Activist scenarios where digital communication leaves forensic traces
- Long-term secure archival where internet won't exist in 50 years

For typical use cases, online encrypted communication (Signal, Briar) offers better usability with comparable security.

## Forensic Considerations

Offline communication with encrypted media creates forensic artifacts that must be managed carefully.

### Secure Media Destruction

After transferring encrypted data via physical media, destroy the media:

```bash
# Secure flash drive destruction
# Option 1: Multiple overwrite passes
shred -vfz -n 7 /dev/usb-device

# Option 2: Physical destruction
# Disassemble flash drive, remove NAND chip, physically destroy with drill press

# Option 3: Thermal destruction
# Incineration in controlled environment (professional service)
```

Simply deleting files leaves forensic traces. Physical destruction is most reliable.

### Communication Log Management

Maintain encrypted archives of important communications:

```bash
# Archive and encrypt old messages
tar czf messages_archive_2024.tar.gz messages/
gpg --symmetric --cipher-algo AES256 messages_archive_2024.tar.gz

# Store archive in multiple locations
# Keep physical backup in fireproof safe

# Document location of backups securely
# In case of death, executor can access if needed
```

### Legal Protection Through Documentation

In jurisdictions with mandatory decryption laws, prepare:

1. **Attorney guidance**: Consult attorney about your jurisdiction's rules
2. **Key recovery procedures**: Document how to prove you cannot access data (e.g., key is destroyed)
3. **Communication logs**: Keep records of what was communicated and when
4. **Forensic artifacts**: Preserve evidence of encryption implementation

This documentation protects you if questioned about encrypted communications.

## When Offline Encryption is Overkill

For most people and scenarios, simpler solutions work:

- **Personal correspondence**: Use Signal or ProtonMail instead
- **File backup**: Use encrypted cloud storage with zero-knowledge provider
- **Collaborative work**: Use XMPP with E2EE or Matrix

Reserve offline encryption for high-threat scenarios where network monitoring is ubiquitous and consequences of exposure are severe.
---


## Frequently Asked Questions

**How long does it take to set up offline encrypted communication between two?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How to Set Up Encrypted Communication for Mutual Aid Network](/privacy-tools-guide/how-to-set-up-encrypted-communication-for-mutual-aid-network/)
- [Briar Messenger Offline Communication](/privacy-tools-guide/briar-messenger-offline-communication-how-it-works-for-prote/)
- [How To Set Up Vpn Failover Between Two Providers Automatical](/privacy-tools-guide/how-to-set-up-vpn-failover-between-two-providers-automatical/)
- [Set Up Secure Communication for Labor Strike Organizing](/privacy-tools-guide/how-to-set-up-secure-communication-for-labor-strike-organizing/)
- [Set Up Secure Communication For Labor Strike Organizing](/privacy-tools-guide/how-to-set-up-secure-communication-for-labor-strike-organizing/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

