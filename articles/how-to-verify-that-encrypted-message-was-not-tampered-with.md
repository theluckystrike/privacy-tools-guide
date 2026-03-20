---
layout: default
title: "How to Verify That Encrypted Message Was Not Tampered With"
description: "A technical guide for developers and power users on verifying encrypted message integrity using HMACs, digital signatures, and authenticated encryption."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-verify-that-encrypted-message-was-not-tampered-with/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Use authenticated encryption (AES-GCM, ChaCha20-Poly1305) which automatically detects tampering by validating an authentication tag—if anyone modifies the ciphertext, decryption fails and the tampering is detected. Alternatively, append an HMAC computed over the encrypted message using a secret key, send both the ciphertext and HMAC, and have the recipient recompute the HMAC to verify it matches. Modern protocols like Signal and TLS use authenticated encryption by default, so for most use cases you inherit this protection automatically.

## Understanding the Integrity Problem

When you encrypt a message, you transform plaintext into ciphertext that unreadable without the decryption key. However, encryption schemes like AES in ECB or CBC mode do not protect against manipulation. An attacker can flip bits in the ciphertext, and the decrypted result will change—potentially in ways that benefit the attacker.

Consider this scenario: you send a banking transaction encoded as JSON. Without integrity verification, an attacker could modify the amount field from "100" to "9999" after encryption. The recipient decrypts the message successfully and processes the altered transaction.

Three primary approaches solve this problem: Message Authentication Codes (MACs), digital signatures, and Authenticated Encryption with Associated Data (AEAD).

## HMAC: Hash-Based Message Authentication

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

## Digital Signatures: Asymmetric Integrity Verification

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

## Authenticated Encryption (AEAD): Integrated Protection

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

## Choosing the Right Approach

For symmetric scenarios where both parties share a secret key, AEAD modes like AES-GCM or ChaCha20-Poly1305 are the recommended choice—they're simpler to implement correctly and faster than combining separate encryption and HMAC steps.

When using asymmetric encryption, apply digital signatures to the encrypted message or use hybrid encryption where you encrypt a symmetric key with the recipient's public key, then sign with your private key.

For legacy systems using unauthenticated encryption modes like CBC, always add HMAC over the ciphertext (not just the plaintext) using a separately derived key.

## Key Security Principles

Never use the same key for encryption and authentication. Derive separate keys using HKDF or a similar key derivation function. Always include the IV or nonce in the authenticated data. Reuse of nonces with AEAD modes completely breaks security—ensure nonces are unique per message.

The verification step is not optional. Always validate integrity before processing the decrypted content. An attacker who can trigger decryption failures may leak information through error messages or timing differences.

By implementing proper integrity verification, you ensure that encrypted messages remain trustworthy—even when transported through hostile networks.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
