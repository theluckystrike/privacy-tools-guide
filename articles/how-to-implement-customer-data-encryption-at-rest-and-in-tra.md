---
layout: default
title: "How To Implement Customer Data Encryption At Rest And In Tra"
description: "A developer guide for implementing encryption at rest and in transit. Learn practical techniques with code examples for securing customer"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-implement-customer-data-encryption-at-rest-and-in-tra/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, encryption]
---

{% raw %}

Encryption serves as your last line of defense when attackers bypass other security controls. Whether you store customer data on disk or transmit it across networks, implementing proper encryption at rest and in transit prevents unauthorized access even if storage media or network traffic gets compromised. This guide walks through practical implementation strategies with working code examples you can adapt for your projects.

## Understanding Encryption at Rest versus in Transit

Encryption at rest protects stored data—when someone gains physical or filesystem access to your servers or databases. Encryption in transit safeguards data moving between client applications and servers, protecting against network sniffing and man-in-the-middle attacks. Both layers work together to create defense in depth, and you should implement both rather than choosing one over the other.

Many compliance frameworks require both protections. The PCI DSS standard explicitly mandates encryption for cardholder data at rest and in transit. GDPR doesn't prescribe specific encryption methods but requires "appropriate technical measures" including encryption for personal data.

## Implementing Encryption at Rest

### Database-Level Encryption

Most modern databases provide transparent encryption that protects data without application code changes. PostgreSQL offers table space encryption and column-level encryption through extensions.

For PostgreSQL with pgcrypto, you can encrypt specific columns:

```sql
-- Create table with encrypted sensitive field
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    encrypted_ssn BYTEA,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Encrypt data using pgcrypto
INSERT INTO customers (email, encrypted_ssn)
VALUES (
    'customer@example.com',
    pgp_sym_encrypt('123-45-6789', 'your-encryption-key')
);

-- Decrypt when needed
SELECT email,
       pgp_sym_decrypt(encrypted_ssn::bytea, 'your-encryption-key') AS ssn
FROM customers;
```

For production systems, avoid storing encryption keys in your application code. Instead, use a key management service (KMS) or environment variables with proper access controls.

### Application-Level Encryption

For finer control, encrypt data within your application before storing. The `cryptography` library in Python provides primitives:

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import base64
import os

def derive_key(password: str, salt: bytes) -> bytes:
    """Derive encryption key from password using PBKDF2."""
    kdf = PBKDF2(
        algorithm=hashes.SHA256(),
        length=32,
        salt=salt,
        iterations=100000,
    )
    return base64.urlsafe_b64encode(kdf.derive(password.encode()))

def encrypt_data(plaintext: str, key: bytes) -> tuple[bytes, bytes]:
    """Encrypt data with a random IV, return (ciphertext, iv)."""
    f = Fernet(key)
    ciphertext = f.encrypt(plaintext.encode())
    return ciphertext

# Generate key from password
salt = os.urandom(16)
key = derive_key("your-secure-password", salt)

# Encrypt sensitive data
encrypted = encrypt_data("customer-ssn-123-45-6789", key)
print(f"Encrypted: {base64.b64encode(encrypted).decode()}")
```

Always use authenticated encryption modes like AES-GCM to prevent tampering. The Fernet library uses AES-CBC with HMAC for authentication, providing both confidentiality and integrity.

### Full Disk Encryption

For protecting entire volumes, use operating system-level encryption. Linux dm-crypt and LUKS provide transparent full-disk encryption:

```bash
# Create encrypted container
sudo cryptsetup luksFormat /dev/sdb1

# Open the encrypted container
sudo cryptsetup luksOpen /dev/sdb1 encrypted_volume

# Create filesystem
sudo mkfs.ext4 /dev/mapper/encrypted_volume

# Mount and use
sudo mount /dev/mapper/encrypted_volume /mnt/protected
```

For cloud environments, enable encryption at rest provided by your cloud provider. AWS EBS, Azure Disk Storage, and Google Cloud Persistent Disk all offer encryption by default using KMS-managed keys.

## Implementing Encryption in Transit

### TLS/SSL Configuration

Transport Layer Security protects data in transit. Configure your web server to use strong TLS versions and cipher suites:

```nginx
# Nginx TLS configuration
server {
    listen 443 ssl http2;

    ssl_certificate /etc/ssl/certs/your-certificate.crt;
    ssl_certificate_key /etc/ssl/private/your-private-key.key;

    # TLS 1.3 only (most secure)
    ssl_protocols TLSv1.3;

    # Strong cipher suites
    ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    # HSTS header
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Additional security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
}
```

For older systems requiring TLS 1.2, disable vulnerable cipher suites and enable only those using AEAD (Authenticated Encryption with Associated Data) modes.

### Certificate Management

Automate certificate issuance and renewal using Let's Encrypt with Certbot:

```bash
# Install Certbot
sudo apt-get install certbot python3-certbot-nginx

# Obtain and install certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Set up automatic renewal
sudo certbot renew --dry-run
```

Consider certificate transparency logging requirements. Modern browsers require certificates to be logged in public certificate transparency logs, which provides visibility into issued certificates.

### Application-Layer Encryption

For highly sensitive data, add encryption even within TLS-protected connections. This protects against compromised TLS certificates or network-level attacks:

```python
import requests
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.encryption import padding
from cryptography.hazmat.primitives import hashes

# Client-side encryption before transmission
def encrypt_for_server(public_key_bytes: bytes, plaintext: bytes) -> bytes:
    public_key = serialization.load_pem_public_key(public_key_bytes)

    # Generate random AES key
    aes_key = os.urandom(32)
    iv = os.urandom(16)

    # Encrypt data with AES
    cipher = Cipher(
        algorithms.AES(aes_key),
        modes.CBC(iv),
        backend=default_backend()
    )
    encryptor = cipher.encryptor()

    # Pad data to block size
    padder = padding.PKCS7(128).padder()
    padded_data = padder.update(plaintext) + padder.finalize()
    ciphertext = encryptor.update(padded_data) + encryptor.finalize()

    # Encrypt AES key with RSA public key
    encrypted_key = public_key.encrypt(
        aes_key,
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    return encrypted_key + iv + ciphertext
```

This approach ensures that even if someone intercepts network traffic or compromises the server, they cannot access the encrypted payload without the private key.

## Key Management Best Practices

Encryption strength depends entirely on key management. Follow these practices:

1. **Never hardcode keys**: Use environment variables, configuration files outside version control, or KMS solutions
2. **Rotate keys regularly**: Implement key rotation schedules (annually for encryption keys, more frequently for signing keys)
3. **Use separate keys for different purposes**: Separate encryption keys from signing keys
4. **Implement key backup and recovery**: Store encrypted backups of keys in secure locations
5. **Monitor key usage**: Log and monitor key access patterns for anomaly detection


## Related Articles

- [Implement Data Minimization Principle in Application Design](/privacy-tools-guide/how-to-implement-data-minimization-principle-in-application-/)
- [Implement Data Portability Feature For Customers Gdpr Right](/privacy-tools-guide/how-to-implement-data-portability-feature-for-customers-gdpr-right-explained/)
- [Implement Purpose Limitation in Data Architecture](/privacy-tools-guide/how-to-implement-purpose-limitation-in-data-architecture-res/)
- [How To Set Up Privacy Focused Crm That Minimizes Customer Da](/privacy-tools-guide/how-to-set-up-privacy-focused-crm-that-minimizes-customer-da/)
- [How To Set Up Privacy Preserving Customer Analytics Without](/privacy-tools-guide/how-to-set-up-privacy-preserving-customer-analytics-without-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
