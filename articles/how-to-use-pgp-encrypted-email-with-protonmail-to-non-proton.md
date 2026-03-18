---

layout: default
title: "How to Use PGP Encrypted Email with ProtonMail to Non-ProtonMail Recipients."
description: "A practical guide for developers and power users on sending PGP-encrypted emails from ProtonMail to external recipients. Includes key management, configuration, and troubleshooting."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-pgp-encrypted-email-with-protonmail-to-non-proton/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

ProtonMail provides built-in PGP support for encrypting emails, but the experience differs significantly when communicating with ProtonMail users versus external recipients using other email providers. This guide covers the technical details developers and power users need to send encrypted emails from ProtonMail to non-ProtonMail recipients.

## Understanding ProtonMail's PGP Architecture

ProtonMail implements PGP encryption at the server level, which means your private keys never leave ProtonMail's encrypted storage. When both sender and recipient use ProtonMail, encryption happens automatically and transparently. However, when sending to external addresses, you need to handle key management manually.

ProtonMail supports two methods for external recipient encryption:

1. **Address Book PGP Keys**: Store recipient public keys in your ProtonMail address book
2. **Manual Encryption**: Copy and paste public keys for each message

## Retrieving Your ProtonMail Public Key

Before sending encrypted emails to external recipients, ensure your public key is accessible. In ProtonMail:

1. Go to **Settings** → **Keys** → **Public Keys**
2. Find your primary key and click **Export Public Key**
3. Save the ASCII-armored block

You can also generate your public key URL for easy sharing:

```
https://api.protonmail.ch/pks/lookup?op=get&search=your-email@protonmail.com
```

This allows external users to retrieve your key programmatically:

```bash
gpg --keyserver hkps://api.protonmail.ch --search-keys your-email@protonmail.com
gpg --keyserver hkps://api.protonmail.ch --recv-keys KEY_ID
```

## Obtaining Recipient Public Keys

External recipients must provide their PGP public keys. Several methods exist for key exchange:

### Method 1: Key Servers

Many users upload keys to public key servers:

```bash
gpg --keyserver keys.openpgp.org --search-keys recipient@example.com
gpg --keyserver keys.openpgp.org --recv-keys RECIPIENT_KEY_ID
```

### Method 2: Direct Request

For sensitive communications, request keys via a verified channel:

```bash
echo "Please send your PGP public key" | gpg --encrypt --armor \
  --recipient recipient@example.com --output request.asc
```

### Method 3: Manual Key Blocks

Recipients can export keys directly:

```bash
gpg --armor --export recipient@example.com > recipient-pubkey.asc
```

## Configuring ProtonMail for External Encryption

Once you have the recipient's public key, store it in ProtonMail:

1. Open **Contacts** in ProtonMail
2. Create or edit the contact
3. Navigate to the **Encryption** tab
4. Paste the public key in the designated field
5. Save the contact

When composing an email to this contact, ProtonMail automatically detects the stored key and offers encryption options.

## Sending Encrypted Emails: Step-by-Step

### Scenario: Sending to a Gmail User

Assume you need to send an encrypted message to `developer@gmail.com` who uses G Suite and has published their key.

**Step 1: Obtain the public key**

```bash
gpg --keyserver keys.openpgp.org --search-keys developer@gmail.com
# Note the key ID from the search results
gpg --keyserver keys.openpgp.org --recv-keys 0x1234567890ABCDEF
```

**Step 2: Export and save to ProtonMail**

```bash
gpg --armor --export 0x1234567890ABCDEF
```

Copy the output and save it to the contact in ProtonMail.

**Step 3: Compose and encrypt**

When composing your email:
- Enter the recipient address
- ProtonMail detects the stored key
- Select **Encrypt** before sending
- The email is encrypted using the recipient's public key

The recipient receives an encrypted email. To decrypt, they need their corresponding private key and passphrase.

## Technical Implementation Details

### Key Verification

For security-critical communications, verify key fingerprints out-of-band:

```bash
gpg --fingerprint recipient@example.com
# Compare the fingerprint via a trusted channel (phone, in-person)
```

### Encryption Process

When ProtonMail encrypts for external recipients, it:

1. Generates a random session key
2. Encrypts the message body with the session key (AES-256)
3. Encrypts the session key with the recipient's public key (RSA or ECIES)
4. Packages both into an encrypted MIME structure

You can verify this structure manually:

```bash
# View encrypted message headers
# Look for: Content-Type: application/pgp-encrypted
#           Content-Type: application/octet-stream (encrypted payload)
```

### Handling Attachments

ProtonMail encrypts attachments alongside the message body. The process:

```
Attachment → Compress → Encrypt (AES-256) → Embed in encrypted message
```

For additional security with large attachments, consider encrypting separately:

```bash
# Encrypt file before attaching
gpg --symmetric --cipher-algo AES256 --output document.pdf.gpg document.pdf
```

## Troubleshooting Common Issues

### Key Not Found

If ProtonMail cannot find a recipient's key:

- Verify the email address matches exactly
- Check if the key was uploaded to a different server
- Request the key directly from the recipient

### Encryption Fails

Common causes:

1. **Expired keys**: Check key expiration with `gpg --list-keys`
2. **Revoked keys**: Verify key status on keyserver
3. **Invalid format**: Ensure proper ASCII-armor formatting

```bash
# Check key validity
gpg --check-signatures recipient@example.com
```

### Decryption Issues

Recipients may encounter problems if:

- Their private key is corrupted
- Passphrase is incorrect
- Key and message were not properly exchanged

## Advanced: Programmatic Integration

For developers building automated systems:

```python
import gnupg

gpg = gnupr.GPG(gnupghome='/path/to/gnupg/home')

# Encrypt for external recipient
def encrypt_for_protonmail_recipient(message, recipient_key_path):
    with open(recipient_key_path, 'r') as key_file:
        recipient_key = key_file.read()
    
    # Import key first
    import_result = gpg.import_keys(recipient_key)
    key_id = import_result.fingerprints[0]
    
    # Encrypt
    encrypted = gpg.encrypt(
        message,
        recipients=[key_id],
        always_trust=True
    )
    
    return str(encrypted)
```

## Security Considerations

When using PGP with external recipients:

- **Key lifecycle**: Regularly rotate keys (annually recommended)
- **Trust model**: Use web of trust or verify fingerprints personally
- **Metadata**: Remember that subject lines and headers remain unencrypted
- **Forward secrecy**: PGP does not provide forward secrecy; consider Signal for real-time messaging

## Summary

Sending PGP-encrypted emails from ProtonMail to non-ProtonMail recipients requires manual key management but provides robust end-to-end encryption. The workflow involves obtaining recipient public keys, storing them in ProtonMail contacts, and selecting encryption when composing messages. For developers, programmatic tools enable automated encryption workflows. While more complex than ProtonMail-internal encryption, this approach ensures your sensitive communications remain private across provider boundaries.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
