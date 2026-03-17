---

layout: default
title: "How to Set Up Offline Encrypted Communication Between Two People"
description: "A practical guide for developers and power users to establish secure offline communication channels using encryption tools, air-gapped systems, and proven protocols."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-offline-encrypted-communication-between-two-pe/
categories: [guides, security, encryption]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

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

## Conclusion

Offline encrypted communication requires more effort than online alternatives, but provides protection against network-based attacks. GPG remains the most accessible option with broad tool support. For developers, modern libraries like libsodium and age offer simpler APIs with strong security guarantees. The key to effective offline security lies not just in choosing the right encryption tool, but in maintaining rigorous operational security throughout the entire communication lifecycle.

Choose your method based on your threat model, technical capability, and the sensitivity of your communications. Test your setup thoroughly before relying on it for sensitive information.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
