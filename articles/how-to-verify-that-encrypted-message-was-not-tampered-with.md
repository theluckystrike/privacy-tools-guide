---
layout: default
title: "How to Verify That Encrypted Message Was Not Tampered"
description: "A technical guide for developers and power users on verifying encrypted message integrity using HMACs, digital signatures, and authenticated encryption"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-verify-that-encrypted-message-was-not-tampered-with/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

Use authenticated encryption (AES-GCM, ChaCha20-Poly1305) which automatically detects tampering by validating an authentication tag—if anyone modifies the ciphertext, decryption fails and the tampering is detected. Alternatively, append an HMAC computed over the encrypted message using a secret key, send both the ciphertext and HMAC, and have the recipient recompute the HMAC to verify it matches. Modern protocols like Signal and TLS use authenticated encryption by default, so for most use cases you inherit this protection automatically.

## Key Takeaways

- **Modern protocols like Signal**: and TLS use authenticated encryption by default, so for most use cases you inherit this protection automatically.
- **Ed25519 is the recommended algorithm for new implementations**: it offers fast verification, small signature sizes, and strong security guarantees.
- **Reuse of nonces with AEAD modes completely breaks security**: ensure nonces are unique per message.
- **Always use `hmac.compare_digest()` for**: timing-safe comparison to prevent timing attacks.
- **This is useful for**: encrypting a message while authenticating metadata like headers or sequence numbers.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand the Integrity Problem

When you encrypt a message, you transform plaintext into ciphertext that unreadable without the decryption key. However, encryption schemes like AES in ECB or CBC mode do not protect against manipulation. An attacker can flip bits in the ciphertext, and the decrypted result will change—potentially in ways that benefit the attacker.

Consider this scenario: you send a banking transaction encoded as JSON. Without integrity verification, an attacker could modify the amount field from "100" to "9999" after encryption. The recipient decrypts the message successfully and processes the altered transaction.

Three primary approaches solve this problem: Message Authentication Codes (MACs), digital signatures, and Authenticated Encryption with Associated Data (AEAD).

### Step 2: HMAC: Hash-Based Message Authentication

HMAC combines a cryptographic hash function with a secret key to produce a tag that verifies both integrity and authenticity. The sender computes HMAC over the message using a shared secret key, then sends both the ciphertext and the HMAC tag. The recipient recomputes the HMAC and compares it to the received tag.

Python implementation using the `hmac` module:

```python
import hmac
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

def encrypt_with_hmac(plaintext: bytes, key: bytes) -> tuple[bytes, bytes]:
    """Encrypt with AES-CBC and append HMAC-SHA256."""
    iv = os.urandom(16)
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    encryptor = cipher.encryptor()

    # PKCS7 padding
    padding_length = 16 - (len(plaintext) % 16)
    plaintext_padded = plaintext + bytes([padding_length] * padding_length)

    ciphertext = encryptor.update(plaintext_padded) + encryptor.finalize()

    # Compute HMAC over IV || ciphertext
    hmac_key = key  # In practice, use a separate key derived via HKDF
    message_to_authenticate = iv + ciphertext
    tag = hmac.new(hmac_key, message_to_authenticate, hashlib.sha256).digest()

    return ciphertext, tag

def decrypt_with_hmac(ciphertext: bytes, tag: bytes, key: bytes, iv: bytes) -> bytes:
    """Verify HMAC then decrypt."""
    hmac_key = key
    message_to_authenticate = iv + ciphertext
    expected_tag = hmac.new(hmac_key, message_to_authenticate, hashlib.sha256).digest()

    if not hmac.compare_digest(expected_tag, tag):
        raise ValueError("HMAC verification failed - message tampered")

    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()
    plaintext_padded = decryptor.update(ciphertext) + decryptor.finalize()

    # Remove padding
    padding_length = plaintext_padded[-1]
    return plaintext_padded[:-padding_length]
```

The critical detail is computing HMAC over the IV concatenated with ciphertext—this prevents attackers from modifying the IV. Always use `hmac.compare_digest()` for timing-safe comparison to prevent timing attacks.

### Step 3: Digital Signatures: Asymmetric Integrity Verification

When parties do not share a secret key, digital signatures provide integrity verification using asymmetric cryptography. The sender signs the message with their private key, and anyone with the corresponding public key can verify the signature.

Ed25519 is the recommended algorithm for new implementations—it offers fast verification, small signature sizes, and strong security guarantees.

```python
from cryptography.hazmat.primitives.asymmetric import ed25519
from cryptography.hazmat.primitives import serialization
import hashlib

# Key generation (typically done once)
private_key = ed25519.Ed25519PrivateKey.generate()
public_key = private_key.public_key()

def sign_message(message: bytes, private_key) -> bytes:
    """Sign message with Ed25519."""
    return private_key.sign(message)

def verify_signature(message: bytes, signature: bytes, public_key) -> bool:
    """Verify Ed25519 signature."""
    try:
        public_key.verify(signature, message)
        return True
    except Exception:
        return False

# Usage
message = b"Transfer $1000 to account 12345"
signature = sign_message(message, private_key)

# Recipient verifies
if verify_signature(message, signature, public_key):
    print("Signature valid - message not tampered")
else:
    print("Signature invalid - message modified")
```

Digital signatures also provide non-repudiation—the sender cannot claim they did not sign the message. This differs from HMAC, where both parties share the same key and either could have produced the tag.

### Step 4: Authenticated Encryption (AEAD): Integrated Protection

AEAD algorithms combine encryption and integrity verification into a single primitive, eliminating the risk of implementing them incorrectly. AES-GCM and ChaCha20-Poly1305 are the standard choices.

AES-GCM provides both confidentiality and integrity using Galois/Counter Mode. It generates an authentication tag automatically during encryption.

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

def encrypt_aead(plaintext: bytes, key: bytes) -> tuple[bytes, bytes]:
    """Encrypt with AES-GCM (includes integrity check)."""
    nonce = os.urandom(12)  # 96-bit nonce
    aesgcm = AESGCM(key)
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)
    return nonce, ciphertext

def decrypt_aead(ciphertext: bytes, nonce: bytes, key: bytes) -> bytes:
    """Decrypt with AES-GCM - raises exception if tampered."""
    aesgcm = AESGCM(key)
    return aesgcm.decrypt(nonce, ciphertext, None)

# Usage
key = os.urandom(32)  # 256-bit key
plaintext = b"Secret message"

nonce, ciphertext = encrypt_aead(plaintext, key)

# Recipient decrypts - raises InvalidTag if tampered
try:
    decrypted = decrypt_aead(ciphertext, nonce, key)
    print(f"Decrypted: {decrypted}")
except Exception as e:
    print(f"Decryption failed - message tampered: {e}")
```

AEAD also supports Additional Authenticated Data (AAD)—information that is authenticated but not encrypted. This is useful for encrypting a message while authenticating metadata like headers or sequence numbers.

```python
# Encrypt with AAD
aesgcm = AESGCM(key)
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data=b"header:1")
```

The recipient must provide the same AAD during decryption; otherwise, authentication fails.

### Step 5: Choose the Right Approach

For symmetric scenarios where both parties share a secret key, AEAD modes like AES-GCM or ChaCha20-Poly1305 are the recommended choice—they're simpler to implement correctly and faster than combining separate encryption and HMAC steps.

When using asymmetric encryption, apply digital signatures to the encrypted message or use hybrid encryption where you encrypt a symmetric key with the recipient's public key, then sign with your private key.

For legacy systems using unauthenticated encryption modes like CBC, always add HMAC over the ciphertext (not just the plaintext) using a separately derived key.

### Step 6: Key Security Principles

Never use the same key for encryption and authentication. Derive separate keys using HKDF or a similar key derivation function. Always include the IV or nonce in the authenticated data. Reuse of nonces with AEAD modes completely breaks security—ensure nonces are unique per message.

The verification step is not optional. Always validate integrity before processing the decrypted content. An attacker who can trigger decryption failures may leak information through error messages or timing differences.

By implementing proper integrity verification, you ensure that encrypted messages remain trustworthy—even when transported through hostile networks.

### Step 7: Real-World Protocol Examples

Understanding how real-world protocols implement integrity verification provides practical insights.

### TLS Record Protection

TLS (Transport Layer Security) uses HMAC to protect encrypted records:

```
TLS Record = [Version][Type][Length][Ciphertext][MAC]
             [2 bytes][1 byte][2 bytes][variable][variable]
```

During decryption:
1. Extract MAC from end of ciphertext
2. Decrypt remaining ciphertext
3. Recompute MAC over decrypted plaintext
4. Compare computed MAC with extracted MAC
5. If they don't match, connection terminates

This approach protects both content and record boundaries from tampering.

### Signal Protocol Message Format

Signal Protocol messages include authentication tags automatically:

```
Signal Frame = [Version][Type][Sender Key ID][Counter][Ciphertext][Auth Tag]
               [1 bit][3 bits][32 bits][32 bits][variable][16 bytes]
```

Each message is authenticated with the recipient's key. If an attacker modifies any byte of the ciphertext, the authentication tag becomes invalid during decryption.

### PGP Signature Integration

For email, PGP combines encryption and digital signatures:

```bash
# Sender encrypts and signs
gpg --encrypt --sign --recipient "recipient@example.com" document.txt

# Results in document.txt.gpg containing:
# 1. Encrypted plaintext
# 2. Signature over plaintext (inside encryption)
# 3. Outer signature (over encrypted content)

# Recipient verifies both integrity and authenticity
gpg --decrypt document.txt.gpg
# Automatically verifies both signatures
```

## Implementation Best Practices

Successful integrity verification requires more than choosing the right algorithm—careful implementation prevents subtle attacks.

### Constant-Time Comparison

Always use constant-time comparison functions to prevent timing attacks:

```python
# BAD: Standard comparison leaks timing information
if computed_tag == received_tag:
    return True
# Time to return depends on where mismatch occurs

# GOOD: Constant-time comparison
import hmac
if hmac.compare_digest(computed_tag, received_tag):
    return True
# Always takes same time regardless of mismatch position
```

Timing attacks are subtle but real. Attackers can measure response times microsecond-by-microsecond to forge valid tags.

### Nonce Management

Reusing nonces with AEAD modes completely breaks security:

```python
# DANGEROUS: Reusing nonce with same key
key = os.urandom(32)
nonce = os.urandom(12)

message1 = aesgcm.encrypt(nonce, "Secret 1", None)
message2 = aesgcm.encrypt(nonce, "Secret 2", None)
# Attacker can now compute: "Secret 1" XOR "Secret 2"
# This reveals relationship between both secrets

# CORRECT: Unique nonce per message
key = os.urandom(32)

nonce1 = os.urandom(12)
message1 = aesgcm.encrypt(nonce1, "Secret 1", None)

nonce2 = os.urandom(12)
message2 = aesgcm.encrypt(nonce2, "Secret 2", None)

# Never reuse nonce with same key
```

Many real-world vulnerabilities stem from nonce reuse. For counter-based nonces, ensure counter never wraps around with the same key.

### Key Derivation

Never use the same key for encryption and authentication:

```python
# Use HKDF to derive separate keys
import hkdf
import hashlib

master_key = os.urandom(32)

# Derive encryption and authentication keys
encrypt_key = hkdf.hkdf_expand(
    hkdf.hkdf_extract(b'', master_key, hashlib.sha256),
    b'encryption',
    32,
    hashlib.sha256
)

auth_key = hkdf.hkdf_expand(
    hkdf.hkdf_extract(b'', master_key, hashlib.sha256),
    b'authentication',
    32,
    hashlib.sha256
)

# Use separate keys for encryption and HMAC
ciphertext = encrypt_with_key(plaintext, encrypt_key)
auth_tag = hmac_with_key(ciphertext, auth_key)
```

Key separation prevents attacks where weaknesses in one operation compromise both.

### Step 8: Common Implementation Mistakes

Real-world systems often make subtle mistakes:

### Mistake 1: Authenticating Only Plaintext

```python
# WRONG: Authenticate plaintext, encrypt result
plaintext = b"Transfer $1000 to account 12345"
tag = hmac.new(key, plaintext, hashlib.sha256).digest()
ciphertext = encrypt(plaintext)
# Send: ciphertext, tag

# PROBLEM: Attacker can swap ciphertexts
# Original: plaintext1 → ciphertext1 (with tag1)
# Attack: ciphertext2 (swapped) with tag1
# Recipient verifies tag over PLAINTEXT after decryption
# But plaintext came from different ciphertext!
```

Always authenticate the ciphertext, not plaintext before encryption.

### Mistake 2: Forgetting to Include IV in Authentication

```python
# WRONG: IV not included in authenticated data
iv = os.urandom(16)
ciphertext = cipher.encrypt(iv, plaintext)
tag = hmac.new(key, ciphertext, hashlib.sha256).digest()
# Send: iv, ciphertext, tag

# PROBLEM: Attacker modifies IV
# Recipient decrypts with modified IV
# Gets different plaintext, but tag is still valid
# (because tag was only over ciphertext)

# CORRECT: Include IV in authentication
iv = os.urandom(16)
ciphertext = cipher.encrypt(iv, plaintext)
tag = hmac.new(key, iv + ciphertext, hashlib.sha256).digest()
# Now modification to IV or ciphertext breaks tag
```

Include all variable components in authenticated data, including IVs and nonces.

### Mistake 3: Accepting Decryption Errors as Integrity Failures

```python
# WRONG: Trust decryption result
try:
    plaintext = aesgcm.decrypt(nonce, ciphertext, None)
    process(plaintext)  # Trust it succeeded
except:
    # Assume tampering
    pass

# PROBLEM: Exception could indicate other errors
# Memory corruption, library bugs, etc.

# CORRECT: Explicit integrity check before processing
try:
    plaintext = aesgcm.decrypt(nonce, ciphertext, None)
except cryptography.hazmat.primitives.ciphers.aead.InvalidTag:
    # Known tampering
    raise
except Exception as e:
    # Unknown error - don't process
    log_and_exit(f"Decryption error: {e}")
```

Distinguish between integrity failures (log, reject) and other errors (fail safely).

### Step 9: Test Your Implementation

Proper testing catches integrity verification bugs before deployment:

```python
def test_integrity_detection():
    """Verify that tampered messages are rejected."""
    key = os.urandom(32)
    plaintext = b"Critical data"

    # Encrypt with AEAD
    nonce, ciphertext = encrypt_aead(plaintext, key)

    # Tamper with ciphertext
    tampered = bytearray(ciphertext)
    tampered[0] ^= 0x01  # Flip single bit

    # Verify tampered message is rejected
    with pytest.raises(ValueError, match="tampered"):
        decrypt_aead(bytes(tampered), nonce, key)

def test_timing_resistance():
    """Verify comparison is timing-resistant."""
    # Placeholder: Real timing tests require nanosecond precision
    # and analysis of timing distributions
    pass
```

Include tampering tests in your test suite. If tampered messages are accepted, your implementation has a critical flaw.
---


## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to verify that encrypted message was not tampered?**

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

- [Use Tcpdump to Verify VPN Traffic Is Encrypted](/privacy-tools-guide/a140-how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [How To Use Tcpdump To Verify Vpn Traffic Is Encrypted](/privacy-tools-guide/how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [How To Verify That Your Encrypted Messages Are Not Being Int](/privacy-tools-guide/how-to-verify-that-your-encrypted-messages-are-not-being-int/)
- [How to Use GPG Signed Emails to Verify Sender Identity](/privacy-tools-guide/how-to-use-gpg-signed-emails-to-verify-sender-identity-step-/)
- [Use GPG Signed Emails to Verify Sender Identity](/privacy-tools-guide/how-to-use-gpg-signed-emails-to-verify-sender-identity/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
