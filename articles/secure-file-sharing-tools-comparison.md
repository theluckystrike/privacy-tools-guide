---
layout: default
title: "Secure File Sharing Tools Comparison: E2E Encrypted."
description: "Compare E2E encrypted file sharing: OnionShare, Tresorit Send, Wormhole, Firefox Send alternatives, Magic Wormhole. CLI examples, size limits, pricing"
date: 2026-03-20
last_modified_at: 2026-03-20
author: "Privacy Tools Guide"
permalink: /secure-file-sharing-tools-comparison/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Traditional file sharing (Dropbox, Google Drive, OneDrive) logs server-side metadata. Even with password protection, service providers see file names, access patterns, and IP addresses. For sensitive documents (medical records, legal files, financial data, NDAs), end-to-end encryption is required.

This guide compares five E2E encrypted file sharing tools: OnionShare, Tresorit Send, Wormhole (Magic Wormhole), Firefox Send alternatives, and open-source options. We focus on security implementation, usability, CLI support, file size limits, and pricing.

## OnionShare

OnionShare is an open-source tool for securely sharing files via Tor. Files are encrypted, hosted locally on your machine, accessible only over Tor with a unique URL and password.

**Security Model:**
- AES-256 encryption (files encrypted before transmission)
- Tor hidden service (no central server, server is your computer)
- No metadata logged (Tor browser hides IP)
- Files deleted after download or timeout (configurable)
- Password-protected sharing link (required)

**Pricing:**
- Free, open source (GPLv3)
- No registration, no accounts

**File Size Limits:**
- Unlimited local sharing (limited by disk space)
- Recommended: < 10 GB per share (Tor bandwidth limited)

**CLI Usage:**

```bash
# Install
brew install onionshare-cli

# Share a file (starts local Tor server)
onionshare-cli --auto-start share /path/to/secret-document.pdf

# Output:
# OnionShare 2.6 | https://[onionaddress].onion/share/[code]
# Password: [random]

# Configure options
onionshare-cli --auto-start \
  --no-slugs \
  --auto-stop \
  --timer 3600 \
  share /path/to/file

# Share multiple files
onionshare-cli --auto-start share file1.pdf file2.docx file3.zip

# Receive files (reverse share)
onionshare-cli --auto-start receive
# Output: https://[onionaddress].onion/receive/[code]
```

**Workflow Example:**

```bash
# Sender
onionshare-cli --auto-start share contract.pdf
# Output: https://abc123def456.onion/share/abc123
# Password: secure_random_password

# Recipient (copy URL and password)
# 1. Open Tor Browser
# 2. Paste URL
# 3. Enter password
# 4. Download file

# Server stops after download (--auto-stop enabled)
```

**Strengths:**
- True anonymity (Tor hidden service)
- No central server risk
- No account creation required
- CLI-friendly
- Receive mode for incoming files

**Weaknesses:**
- Requires Tor Browser on recipient side
- Slow transfer over Tor (typical: 500 KB/s)
- Sharing link only works while sender's computer is on
- No cloud backup (you are the server)
- Password sharing over separate channel required

---

## Tresorit Send

Tresorit Send is a commercial E2E encrypted file sharing service owned by Tresorit (a zero-knowledge cloud provider). Files are encrypted client-side, uploaded to Tresorit servers, encrypted link generated.

**Security Model:**
- AES-256 encryption (client-side before upload)
- Zero-knowledge architecture (Tresorit doesn't hold encryption keys)
- Recipient must decrypt on download
- Expiration dates enforced
- Optional password protection
- Download activity logged to sender only

**Pricing:**
- Free: 5 GB/month, 7-day expiration
- Tresorit Plus: $99/year (100 GB/month, 90-day expiration, custom branding)
- Tresorit Business: Custom (team accounts, audit logs, admin controls)

**File Size Limits:**
- Free: 2 GB per file, 5 GB total per month
- Plus: 10 GB per file, 100 GB total per month

**CLI Usage:**
No native CLI. Web API available for integrations.

**Web Workflow:**

```
1. Visit tresorit.com/send
2. Upload file (encrypted client-side)
3. Set expiration (1, 7, 30, 90 days)
4. Optional: Add password, set download limit
5. Get shareable link (https://send.tresorit.com/[id])
6. Send link + password separately
```

**Browser Extension Workflow (Faster):**

```
1. Right-click file in file explorer
2. Select "Send with Tresorit"
3. Configure expiration/password
4. Link automatically copied to clipboard
5. Paste in chat/email
```

**Strengths:**
- User-friendly web interface
- No setup required
- Reliable (commercial service)
- Browser extension support
- Zero-knowledge confirmed by Tresorit
- Download tracking (sender sees when received)

**Weaknesses:**
- No CLI, limited automation
- Proprietary (not open source)
- Requires Tresorit account
- API rate limits for bulk sharing
- Pricing adds up for frequent sharing

---

## Wormhole (Magic Wormhole)

Wormhole is an open-source CLI tool for secure file transfer using encryption and the Rendezvous protocol. Files are encrypted with a passphrase, transferred directly between sender and recipient.

**Security Model:**
- SPAKE2 key exchange (deriving shared encryption key from passphrase)
- Salsa20-Poly1305 encryption (authenticated encryption)
- Files transferred peer-to-peer (no central server stores files)
- Passphrase shared out-of-band (phone, chat)
- Transit relay optional (NAT traversal)

**Pricing:**
- Free, open source (MIT)
- No accounts, no registration

**File Size Limits:**
- Unlimited (limited by network and disk)
- Passphrase-based transfer works best for < 10 GB

**CLI Usage:**

```bash
# Install
brew install magic-wormhole

# Send a file
wormhole send /path/to/secret.pdf

# Output:
# Sending 1.2 MB file named 'secret.pdf'
# On the other computer, please run: wormhole receive
# Wormhole code is: 7-saturn-giddy
# (Run Ctrl-C to cancel)

# Recipient (on different machine)
wormhole receive

# Output:
# Enter receive wormhole code: 7-saturn-giddy
# Receiving file (1.2 MB) named 'secret.pdf' from sender
# ok? (Y/n): Y
# Receiving (1.2 MB)...............................
# Received file written to ./secret.pdf
```

**Advanced Examples:**

```bash
# Send and specify filename
wormhole send --text "my secret data"
# Output: Wormhole code: 7-saturn-giddy

# Receive text instead of file
wormhole receive 7-saturn-giddy

# Send with custom passphrase (not recommended)
# wormhole send --code="sunny-machine" /file.pdf

# Use specific transit relay (for NAT issues)
wormhole --transit-relay ws://relay.example.com send /file.pdf

# Batch transfer (directory)
wormhole send /path/to/directory/
# Automatically zips and transfers as single file
```

**Workflow Example:**

```
Sender (Terminal 1):
$ wormhole send contract.pdf
Sending 0.3 MB file named 'contract.pdf'
On the other computer, please run: wormhole receive
Wormhole code is: 3-boil-muffin
(Run Ctrl-C to cancel)

Recipient (Terminal 2):
$ wormhole receive
Enter receive wormhole code: 3-boil-muffin
Receiving file (0.3 MB) named 'contract.pdf' from sender
ok? (Y/n): Y
Received file written to ./contract.pdf

Sender:
All done; waiting for the next one. (Run Ctrl-C to cancel)
```

**Strengths:**
- Minimal: 1 passphrase (7-word code) to share
- Peer-to-peer (no file server needed)
- CLI-native (ideal for automation)
- Open source
- No account creation
- Fast (limited only by network)

**Weaknesses:**
- Both parties must run Wormhole simultaneously (send/receive)
- Passphrase-based (code valid only during transfer)
- Limited to command line (no GUI)
- NAT traversal requires transit relay
- File deletion not enforced on recipient side

---

## Firefox Send Alternatives

Firefox Send shut down in 2020, but similar E2E encrypted services emerged. We compare top alternatives.

**Best Alternative: CryptDrop**

CryptDrop is a privacy-focused file sharing service (uses Mozilla's older Firefox Send code).

**Security:**
- AES-GCM encryption (client-side)
- PBKDF2 key derivation from password
- Files deleted after download or expiration
- No server-side logs

**Pricing:**
- Free (limited features)
- Premium: €5/month

**File Size:**
- Free: 500 MB
- Premium: 20 GB

**Usage:**

```
1. Visit cryptdrop.org
2. Select file (encrypted client-side)
3. Set password, expiration, download limit
4. Get link
5. Share link, password separately
```

**Other Firefox Send Alternatives:**

| Service | Provider | Encryption | Size Limit | Price |
|---|---|---|---|---|
| **CryptDrop** | Cryptpad Labs | AES-GCM | 500 MB free, 20 GB paid | Free/$5/mo |
| **SnapDrop** | Robin Linus | E2E (peer-to-peer) | Unlimited (LAN) | Free |
| **FilePizza** | Scalabull | WebRTC (P2P) | Unlimited (browser memory) | Free |
| **Transfer.sh** | [Community] | Client-side encryption (optional) | 20 GB | Free/$15/mo |

**Transfer.sh Example (CLI):**

```bash
# Upload with encryption (client-side)
curl --upload-file ./document.pdf https://transfer.sh/document.pdf
# Output: https://transfer.sh/[id]/document.pdf

# Delete after 14 days (default)
# Or specify retention:
curl --upload-file ./document.pdf https://transfer.sh/document.pdf?expire=1440
# Expires in 24 hours (1440 minutes)
```

---

## Comparison Table

| Tool | Encryption | Server | Price | Size Limit | CLI | Speed |
|---|---|---|---|---|---|---|
| **OnionShare** | AES-256 | Local (Tor) | Free | Unlimited | Yes | Slow (500KB/s) |
| **Tresorit Send** | AES-256 | Cloud (zero-knowledge) | Free/$99yr | 2GB free, 10GB paid | No | Fast (10+ MB/s) |
| **Wormhole** | Salsa20-Poly1305 | P2P | Free | Unlimited | Yes | Fast (50+ MB/s) |
| **CryptDrop** | AES-GCM | Cloud | Free/$5mo | 500MB free, 20GB paid | No | Fast (10+ MB/s) |
| **Transfer.sh** | Optional | Cloud | Free/$15mo | 20 GB | Yes | Fast (50+ MB/s) |

---

## Use Case Recommendations

**Legal/Confidential Documents:**
OnionShare (true anonymity) or Wormhole (fast, no central server)

**Medical Records:**
Tresorit Send (audit logs, compliance features) or CryptDrop (minimal privacy risk)

**Technical Collaboration (logs, configs):**
Wormhole (CLI-friendly, easy automation)

**One-off Sharing:**
OnionShare (no setup) or Tresorit Send (web interface)

**Highly Sensitive (foreign governments, journalists):**
OnionShare only (Tor hidden service)

---

## Security Checklist

Before using any file sharing service:

- [ ] Does the service support E2E encryption (client-side)?
- [ ] Is encryption open source (auditable)?
- [ ] Are files deleted after download or expiration?
- [ ] Is server-side logging minimal?
- [ ] Can you password-protect or limit downloads?
- [ ] Does the provider have a zero-knowledge privacy policy?
- [ ] Are source code and security audits published?

---

## Practical Workflow Examples

**Scenario 1: Lawyer sharing legal document with client**

Option 1 (Most Secure):
```bash
# Lawyer's computer
onionshare-cli --auto-start --timer 86400 share contract.pdf
# Gives: URL + password

# Phone call: "Go to [URL], password is [word-word-word]"
# Client downloads via Tor Browser
# Document deleted after 24 hours or download
```

Option 2 (More Convenient):
```
1. Visit tresorit.com/send
2. Upload contract.pdf
3. Set expiration to 7 days
4. Set optional password
5. Email link with password in separate message
```

**Scenario 2: Developer sharing config file with remote team**

```bash
# Option 1: Wormhole (fastest, no setup)
wormhole send config.yml
# Sends code to team member
# Team member runs: wormhole receive [code]

# Option 2: Transfer.sh (one-liner)
curl --upload-file config.yml https://transfer.sh/config.yml
# Returns URL for sharing
```

**Scenario 3: Journalist sharing source files with editor (ultra-confidential)**

```bash
# OnionShare only (most anonymous)
onionshare-cli --auto-start --no-slugs share leaked-docs.zip
# Tor-only access, no logs, no account needed
```

---

## Key Recommendation

**For maximum privacy:** OnionShare (Tor, local server, no registration)

**For convenience:** Tresorit Send (web interface, reliable)

**For power users:** Wormhole (CLI, fast, flexible)

**For compliance:** Tresorit Send (audit logs, retention policies)

**For one-off sharing:** CryptDrop or Transfer.sh (minimal setup)

Start with Wormhole if you're technical. Start with Tresorit Send if you prioritize ease of use. Upgrade to OnionShare for legally sensitive documents.


## Related Articles

- [Best Secure File Sharing Tools for Teams Handling.](/privacy-tools-guide/best-secure-file-sharing-tools-for-teams-handling-sensitive-data/)
- [How to Set Up Secure File Sharing for Sensitive Documents](/privacy-tools-guide/how-to-set-up-secure-file-sharing-for-sensitive-documents/)
- [How To Use Age Encryption For Secure File Sharing Command Li](/privacy-tools-guide/how-to-use-age-encryption-for-secure-file-sharing-command-li/)
- [Onionshare Secure File Sharing Over Tor Network Setup And Us](/privacy-tools-guide/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Best Encrypted File Sharing Service 2026](/privacy-tools-guide/best-encrypted-file-sharing-service-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
