---
layout: default
title: "How to Audit Your Cloud Storage Privacy"
description: "Systematically audit what your cloud storage provider stores, shares, and exposes — covering access logs, third-party data sharing, encryption gaps, and legal exposure"
date: 2026-03-22
author: theluckystrike
permalink: /audit-cloud-storage-privacy-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Audit Your Cloud Storage Privacy

Most people pick a cloud storage provider based on price and storage size, then forget about it. The provider can read your files (unless zero-knowledge encrypted), scan them for content, share access logs with advertisers, and hand over data to law enforcement. An audit makes this concrete — what does your provider actually know?

---

## What Cloud Providers Can Access

| Provider type | What they can see |
|---------------|------------------|
| Standard (Dropbox, Box, OneDrive) | All your files, metadata, previews |
| E2EE optional (iCloud with Advanced) | Files if E2EE enabled; metadata always visible |
| Zero-knowledge (Tresorit, ProtonDrive) | Encrypted blobs only, no file content |
| Self-hosted (Nextcloud) | Whatever your server admin has access to |

Even zero-knowledge providers can see: file names (unless encrypted), timestamps, file sizes, IP address, and account data.

---

## Step 1: Review the Privacy Policy

Key questions:

```
1. Do you scan file content?
   - For malware? (common)
   - For advertising?

2. Who do you share data with?
   - "Analytics partners" = advertising ecosystem
   - "Service providers" = subprocessors

3. Law enforcement response:
   - Warrant required?
   - Transparency report published?
   - Do they notify you of requests?

4. Retention policy:
   - How long after deletion?
   - Backup copies — how many, retained how long?

5. Encryption:
   - At rest? (standard, they hold keys)
   - Client-side / zero-knowledge?
```

---

## Step 2: Request Your Data Export

```bash
# Google Drive:
# takeout.google.com → Select Drive → Export
# Includes: files, metadata (created/modified dates, sharing settings)

# Dropbox:
# dropbox.com → Settings → Privacy → Download your data
# Includes: file list, sharing events, linked apps, location of accesses

# OneDrive:
# account.microsoft.com → Privacy → Download your data

# iCloud:
# privacy.apple.com → Request a copy of your data
```

Analyze the metadata:

```python
#!/usr/bin/env python3
"""Analyze Google Takeout export for privacy-relevant metadata."""
import json
from pathlib import Path

def analyze_google_export(export_dir):
    metadata_files = list(Path(export_dir).rglob("*.json"))
    shared_files = []
    third_party_apps = set()

    for mf in metadata_files:
        try:
            data = json.loads(mf.read_text())
            if data.get('shared'):
                shared_files.append(mf.stem)
            for perm in data.get('permissions', []):
                if perm.get('role') == 'reader':
                    third_party_apps.add(perm.get('displayName', 'Unknown'))
        except (json.JSONDecodeError, KeyError):
            pass

    print(f"Shared files: {len(shared_files)}")
    print(f"Third-party access: {third_party_apps}")

analyze_google_export("/path/to/takeout")
```

---

## Step 3: Audit Connected Apps and OAuth Grants

```bash
# Google Drive connected apps:
# myaccount.google.com → Security → Third-party apps with account access
# Revoke anything you don't recognize or no longer use

# Dropbox:
# dropbox.com → Settings → Connected apps
# Shows last used date — revoke stale access

# OneDrive:
# myapps.microsoft.com
```

---

## Step 4: Check Encryption Status

```bash
# Verify TLS on provider endpoints
openssl s_client -connect api.dropbox.com:443 -brief 2>/dev/null | head -5
# Look for: Protocol: TLSv1.3

# For providers claiming zero-knowledge:
# Inspect network traffic with mitmproxy
mitmproxy --mode transparent
# If files are readable as plaintext → NOT zero-knowledge
```

---

## Step 5: Review Shared Links

Old shared links remain active indefinitely — a common exposure point:

```python
import dropbox

dbx = dropbox.Dropbox('YOUR_API_TOKEN')
links = dbx.sharing_list_shared_links()
print(f"Active shared links: {len(links.links)}")

for link in links.links:
    print(f"  {link.path_lower}")
    print(f"  URL: {link.url}")
```

---

## Step 6: Check Data Residency and Transparency Reports

```bash
# Find where your cloud provider's servers are located
host api.dropbox.com
whois $(dig +short api.dropbox.com | head -1)

# Check transparency reports:
# Dropbox: dropbox.com/transparency
# Google: transparencyreport.google.com
# Apple: apple.com/legal/transparency
# Proton: proton.me/legal/transparency
```

What to look at:
- Number of government requests received
- Percentage of requests where data was provided
- Whether they notify users when legally permitted
- NSL count

---

## Remediation If You Find Problems

```bash
# Option 1: Encrypt before upload (Cryptomator)
# Files encrypted locally, upload encrypted blobs to any provider

# Option 2: Migrate to zero-knowledge provider
# Proton Drive, Tresorit, Filen.io

# Option 3: Self-host (Nextcloud)

# Option 4: Rclone with client-side encryption
rclone config
# Set up encrypted remote:
# Type: crypt
# Remote: your existing remote (e.g., dropbox:/)
# Password: set strong password
rclone copy /local/files encrypteddropbox:backup/
```

---

## Related Reading

- [Best Zero Knowledge Cloud Storage 2026](/privacy-tools-guide/best-zero-knowledge-cloud-storage-2026/)
- [Encrypted Cloud Storage Comparison 2026](/privacy-tools-guide/encrypted-cloud-storage-comparison-2026/)
- [Secure Cloud Storage Encryption Rclone](/privacy-tools-guide/secure-cloud-storage-encryption-rclone/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
