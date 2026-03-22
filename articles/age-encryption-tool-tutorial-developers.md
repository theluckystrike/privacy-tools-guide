---
layout: default
title: "Age Encryption Tool Tutorial for Developers"
description: "A practical guide to using age encryption for developers and power users. Learn command-line usage, programmatic integration, and advanced workflows"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /age-encryption-tool-tutorial-developers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, tutorial, encryption]
---
---
layout: default
title: "Age Encryption Tool Tutorial for Developers"
description: "A practical guide to using age encryption for developers and power users. Learn command-line usage, programmatic integration, and advanced workflows"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /age-encryption-tool-tutorial-developers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, tutorial, encryption]
---

{% raw %}

Install age with `brew install age` (macOS) or `go install filippo.io/age@latest`, generate a key pair with `age-keygen`, then encrypt any file with `age -r <public-key> -o output.age input.txt`. Age is a modern, minimal alternative to PGP that handles file encryption with far less complexity -- no key servers, no web of trust, no configuration files. This tutorial covers command-line usage, passphrase-based encryption, Go library integration, SSH key interoperability, and CI/CD pipeline automation.

## Key Takeaways

- **This approach uses your**: existing SSH infrastructure.
- **Use hardware security modules**: or secure key management services for production secrets.
- **When using `-p`**: choose passphrases that meet modern complexity requirements.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Consider a security review**: if your application handles sensitive user data.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install Age

Age supports multiple platforms including macOS, Linux, and Windows. Install it using Homebrew on macOS:

```bash
brew install age
```

On Linux, download the appropriate binary from the GitHub releases page:

```bash
wget https://github.com/FiloSottile/age/releases/download/v1.2.1/age-v1.2.1-linux-amd64.tar.gz
tar -xzf age-v1.2.1-linux-amd64.tar.gz
sudo mv age/age* /usr/local/bin/
```

For Go developers, install directly via go install:

```bash
go install filippo.io/age@latest
```

Verify the installation:

```bash
age --version
age v1.2.1
```

### Step 2: Understand Age's Key Pairs

Age uses two types of keys: identity keys (private keys) and recipient keys (public keys). Generate a new identity key pair:

```bash
# Generate identity (private key)
age-keygen

# Output example:
# # created: 2026-03-15T12:00:00Z
# # public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrjwfkg5vwveyq6mk9l
# AGE-SECRET-KEY-1XK7Q6Z9J8V4Y2W5N3M1L6P0R8T3U9V7W4X1Y2Z5A3B6C9D0E2F4G7H8J
```

Save this output securely. The public key (`age1ql3z...`) allows others to encrypt files for you without accessing your private key.

### Step 3: Encrypt Files

Encrypt a file for yourself using your public key:

```bash
age -p -o encrypted.tar.gz.age secret.tar.gz
```

The `-p` flag prompts for a passphrase to add an additional layer of encryption (symmetric encryption layered on top of recipient encryption). The `-o` flag specifies the output file.

Alternatively, encrypt directly with a recipient's public key:

```bash
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrjwfkg5vwveyq6mk9l \
    -o document.pdf.age document.pdf
```

For multiple recipients, specify each public key with a separate `-r` flag:

```bash
age -r age1recipient1... -r age1recipient2... -o shared.tar.gz.age secret.tar.gz
```

### Step 4: Decrypt Files

Decrypt a file using your identity key:

```bash
age -d -o decrypted.tar.gz encrypted.tar.gz.age
```

If the file was encrypted with a passphrase only (using `-p` without `-r`):

```bash
age -d encrypted.tar.gz.age
```

This prompts for the passphrase. For automated workflows, pipe the passphrase:

```bash
echo "your-passphrase" | age -d -i ~/age-key.txt encrypted.tar.gz.age -o decrypted.tar.gz
```

### Step 5: Use Passphrases Only

For quick symmetric encryption without key pairs, use passphrase-based encryption:

```bash
age -p secret.txt > secret.txt.age
```

This creates a file encrypted with a passphrase. Decrypt using the same approach:

```bash
age -d secret.txt.age > restored_secret.txt
```

### Step 6: Implement Programmatic Integration

Integrate age into your Go applications using the age package:

```go
package main

import (
    "bytes"
    "fmt"
    "filippo.io/age"
    "filippo.io/age/armor"
    "io/ioutil"
)

func main() {
    // Generate a new identity
    identity, err := age.GenerateX25519Identity()
    if err != nil {
        panic(err)
    }

    fmt.Println("Public Key:", identity.PublicKey().String())

    // Encrypt data
    plaintext := []byte("Sensitive data to encrypt")

    // Create recipient from public key
    recipient, err := age.ParseX25519Recipient(identity.PublicKey().String())
    if err != nil {
        panic(err)
    }

    var buf bytes.Buffer
    w, err := age.Encrypt(&buf, recipient)
    if err != nil {
        panic(err)
    }

    if _, err := w.Write(plaintext); err != nil {
        panic(err)
    }
    if err := w.Close(); err != nil {
        panic(err)
    }

    // Decrypt data
    r, err := age.Decrypt(bytes.NewReader(buf.Bytes()), identity)
    if err != nil {
        panic(err)
    }

    decrypted, err := ioutil.ReadAll(r)
    if err != nil {
        panic(err)
    }

    fmt.Println("Decrypted:", string(decrypted))
}
```

Install the package:

```bash
go get filippo.io/age
```

### Step 7: Integration with SSH Keys

Age can derive recipient keys from existing SSH keys, making migration easier:

```bash
# Generate age key from SSH key
ssh-to-age < ~/.ssh/id_rsa.pub
```

Use the resulting age public key for encryption. This approach uses your existing SSH infrastructure.

### Step 8: Create Shell Script Automation

Create a reusable encryption script:

```bash
#!/bin/bash
# encrypt.sh

RECIPIENT="$1"
INPUT_FILE="$2"
OUTPUT_FILE="${INPUT_FILE}.age"

if [ -z "$RECIPIENT" ] || [ -z "$INPUT_FILE" ]; then
    echo "Usage: $0 <recipient-public-key> <file-to-encrypt>"
    exit 1
fi

age -r "$RECIPIENT" -o "$OUTPUT_FILE" "$INPUT_FILE"
echo "Encrypted: $OUTPUT_FILE"
```

Create a corresponding decryption script:

```bash
#!/bin/bash
# decrypt.sh

INPUT_FILE="$1"
OUTPUT_FILE="${INPUT_FILE%.age}"

if [ -z "$INPUT_FILE" ]; then
    echo "Usage: $0 <file-to-decrypt>"
    exit 1
}

age -d -o "$OUTPUT_FILE" "$INPUT_FILE"
echo "Decrypted: $OUTPUT_FILE"
```

Make them executable and use them:

```bash
chmod +x encrypt.sh decrypt.sh
./encrypt.sh age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrjwfkg5vwveyq6mk9l backup.tar.gz
./decrypt.sh backup.tar.gz.age
```

### Step 9: Configure CI/CD Pipeline Integration

Use age in automated deployment pipelines. Generate keys specifically for CI environments:

```bash
# In your CI pipeline
age-keygen > /tmp/ci-age-key.txt

# Encrypt secrets for the pipeline
age -r "$(grep 'public key:' /tmp/ci-age-key.txt | awk '{print $3}')" \
    -o secrets.age .env

# Decrypt during build
age -d -i /tmp/ci-age-key.txt -o .env secrets.age
```

Store the CI identity key as a secret in your CI platform's secrets management.

## Security Considerations

When using age in production environments, follow these practices:

Store identity keys separately from encrypted data. Use hardware security modules or secure key management services for production secrets.

Periodically rotate keys and re-encrypt sensitive data. Age makes this straightforward—decrypt with the old key and re-encrypt with a new one.

When using `-p`, choose passphrases that meet modern complexity requirements. Combine with public-key encryption for defense in depth.

Log encryption and decryption operations in sensitive environments. Consider adding metadata to encrypted files for organizational purposes.

## Troubleshooting

Common issues and solutions:

If you get a permission denied error, ensure your identity key file has restrictive permissions:

```bash
chmod 600 ~/.age/identity
```

If you see an invalid key format error, verify the public key format—age public keys start with `age1`. Check for copy errors when sharing keys.

A passphrase mismatch occurs when decrypting files encrypted with `-p` using the wrong passphrase. There is no recovery mechanism for lost passphrases.

Age encrypts files in streaming fashion, handling large files efficiently. For very large files, consider splitting them first with `split` before encryption.

## Frequently Asked Questions

**How long does it take to developers?**

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

- [How To Use Age Encryption For Secure File Sharing Command](/privacy-tools-guide/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
- [How to Set Up age Encryption for Teams](/privacy-tools-guide/age-encryption-team-setup-guide/)
- [How to Use Age Encryption for File Sharing 2026](/privacy-tools-guide/how-to-use-age-encryption-for-file-sharing-2026/)
- [How to Encrypt Git Repos with git-crypt and age](/privacy-tools-guide/how-to-encrypt-git-repos-with-git-crypt-and-age/)
- [How To Implement Customer Data Encryption At Rest](/privacy-tools-guide/how-to-implement-customer-data-encryption-at-rest-and-in-tra/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
