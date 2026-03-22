---
layout: default

permalink: /restic-vs-borg-backup-encrypted-comparison/
description: "Compare restic vs borg backup encrypted comparison with practical examples, tips, and step-by-step instructions for getting the best results."
tags: [privacy-tools-guide, comparison]
author: "Privacy Tools Guide"
reviewed: true
score: 8
date: 2026-03-15
categories: [guides]
---

layout: default
title: "Restic vs Borg Backup: Encrypted Comparison for Developers"
description: "A technical comparison of Restic and Borg backup tools focusing on encryption, deduplication, and performance for developers and power users"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /restic-vs-borg-backup-encrypted-comparison/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, comparison]
---

{% raw %}

When selecting a backup solution for sensitive data, encryption becomes the deciding factor. Restic and Borg are both open-source, deduplicating backup programs, but their encryption approaches differ significantly. This comparison examines how each tool handles encryption, performance, and usability from a developer's perspective.

## Table of Contents

- [Encryption Architecture](#encryption-architecture)
- [Deduplication and Storage Efficiency](#deduplication-and-storage-efficiency)
- [Performance Benchmarks](#performance-benchmarks)
- [Repository Management](#repository-management)
- [Remote and Cloud Backends](#remote-and-cloud-backends)
- [Automation and Scheduling](#automation-and-scheduling)
- [Key Management: The Most Overlooked Risk](#key-management-the-most-overlooked-risk)
- [Compression Options](#compression-options)
- [Use Case Recommendations](#use-case-recommendations)
- [Security Considerations](#security-considerations)

## Encryption Architecture

### Restic's Encryption

Restic uses **AES-256-CTR** for encryption combined with **Poly1305-AES** for authentication. Every snapshot is encrypted with a repository key derived from your password using **Argon2id** (with scrypt as a fallback). The key derivation uses 64 MB of memory by default, making brute-force attacks computationally expensive.

Initialize an encrypted repository with:

```bash
restic init --repo /backup/restic-repo
# Enter password when prompted

# Or specify repository via environment variable
export RESTIC_PASSWORD="your-secure-password"
restic -r /backup/restic-repo init
```

Restic encrypts data in 1 MB chunks before storage. Each chunk gets a unique ID, and deduplication happens at the encrypted chunk level. This means identical files across different machines will deduplicate correctly without exposing plaintext.

### Borg's Encryption

Borg offers two encryption modes: **AES-OCB** (default, faster) and **ChaCha20-Poly1305** (modern, recommended). Both use **Argon2id** for key derivation with configurable memory and iteration parameters.

Create an encrypted Borg repository:

```bash
# Interactive mode
 borg init --encryption=repokey /backup/borg-repo

# Or with environment variable
export BORG_PASSPHRASE="your-secure-password"
borg init --encryption=repokey-aesomact /backup/borg-repo
```

Borg's `repokey` mode stores the encrypted key on the repository server. For maximum security, use `keyfile` mode where the key stays local only:

```bash
borg init --encryption=keyfile-chacha20poly1305 /backup/borg-repo
```

## Deduplication and Storage Efficiency

Both tools excel at deduplication, but the mechanisms differ.

### Restic's Approach

Restic performs deduplication across all snapshots globally. The 1 MB chunk size optimizes for typical file sizes, and content-defined chunking handles files with small modifications efficiently.

```bash
# Backup a directory
restic -r /backup/restic-repo backup ~/projects/myapp

# Create another backup with changes
restic -r /backup/restic-repo backup ~/projects/myapp

# Check storage usage
restic -r /backup/restic-repo stats
```

Restic reports deduplication savings clearly, showing how much storage deduplication saves compared to raw data.

### Borg's Approach

Borg uses **architectural deduplication**—chunks are indexed by content hash, so identical data deduplicates across all archives in the repository. Borg's chunker uses a rolling hash (Buzhash) that adapts to data patterns.

```bash
# First backup
borg create /backup/borg-repo::archive-1 ~/projects/myapp

# Second backup (deduplicated automatically)
borg create /backup/borg-repo::archive-2 ~/projects/myapp

# List archives and their sizes
borg list /backup/borg-repo
borg info /backup/borg-repo
```

Borg's `info` command shows compression ratios and deduplication statistics, useful for evaluating storage efficiency.

## Performance Benchmarks

Performance depends on CPU for encryption/decryption and I/O for storage. Here's a practical comparison based on typical workloads:

| Aspect | Restic | Borg |
|--------|--------|------|
| First backup | Faster initial hash computation | Slower chunking phase |
| Subsequent backups | Quicker with large unchanged files | Very fast due to chunk cache |
| Restore speed | Comparable | Slightly faster with local repo |
| Memory usage | Lower (~100MB baseline) | Higher (~500MB for large repos) |
| CPU encryption | AES-NI accelerated | AES-NI or SIMD (ChaCha20) |

Run your own benchmark:

```bash
# Restic timing
time restic -r /backup/restic-repo backup ~/projects/myapp

# Borg timing
time borg create /backup/borg-repo::$(date +%Y%m%d) ~/projects/myapp
```

## Repository Management

### Restic

Restic repositories are simpler but less feature-rich:

```bash
# Check repository integrity
restic -r /backup/restic-repo check

# Prune old snapshots (remove unreferenced data)
restic -r /backup/restic-repo prune --keep-last 7

# Forget old snapshots based on retention policy
restic -r /backup/restic-repo forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6
```

Combine prune and forget in one command:

```bash
restic -r /backup/restic-repo forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

### Borg

Borg offers more granular control:

```bash
# Check repository integrity
borg check /backup/borg-repo

# Prune archives
borg prune /backup/borg-repo --keep-daily=7 --keep-weekly=4 --keep-monthly=6

# Compact storage (free space after prune)
borg compact /backup/borg-repo
```

Borg also supports **verification** that actually restores and checks file integrity:

```bash
# Verify archive integrity by extracting and checking
borg verify /backup/borg-repo::archive-1
```

## Remote and Cloud Backends

The choice of storage backend is a significant differentiator in real-world deployments.

**Restic** was designed from day one with remote backends as a first-class feature. Supported backends include local filesystem, SFTP, REST server, Amazon S3, Backblaze B2, Microsoft Azure Blob Storage, and Google Cloud Storage. The REST server backend is particularly useful for self-hosted deployments — you can run restic's official server on any VPS and point multiple clients at it:

```bash
# Start the restic REST server
rest-server --path /data/restic --no-auth --listen :8000

# Initialize remote repository
restic -r rest:http://backup-server:8000/myrepo init

# Back up to remote
restic -r rest:http://backup-server:8000/myrepo backup ~/projects
```

Because all encryption happens client-side before data is transmitted, the server never sees plaintext regardless of whether the connection uses TLS.

**Borg** was built primarily around local and SSH (SFTP) backends. BorgBase, a commercial hosting service, provides managed Borg repositories with append-only mode — a feature where backup clients can add new archives but cannot delete existing ones, protecting against ransomware that gains access to backup credentials. This is a meaningful security property for production backup systems:

```bash
# Initialize append-only Borg repository on BorgBase
borg init --encryption=keyfile-chacha20poly1305 \
  ssh://xxxxxxxxxx@xxxxxxxxxx.repo.borgbase.com/./repo

# Create archive (append-only mode prevents deletion from client)
borg create ssh://xxxxxxxxxx@xxxxxxxxxx.repo.borgbase.com/./repo::$(date +%Y%m%d-%H%M%S) ~/data
```

Restic also supports append-only semantics via repository locks, but BorgBase's server-enforced model is more strong against a compromised client.

## Automation and Scheduling

### Systemd Timer for Restic

```ini
# /etc/systemd/system/restic-backup.service
[Unit]
Description=Restic backup

[Service]
Type=oneshot
EnvironmentFile=/etc/restic/env
ExecStart=/usr/bin/restic -r ${RESTIC_REPOSITORY} backup /home /etc
ExecStartPost=/usr/bin/restic -r ${RESTIC_REPOSITORY} forget --keep-daily 7 --keep-weekly 4 --prune
```

```ini
# /etc/systemd/system/restic-backup.timer
[Unit]
Description=Daily Restic backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

### Cron with Borg and Healthchecks.io

Pairing Borg with a dead man's switch service provides alerting when backups stop running:

```bash
#!/bin/bash
# /usr/local/bin/borg-backup.sh
REPO="ssh://backup@nas.local/./repos/workstation"
export BORG_PASSPHRASE="$(cat /etc/borg/passphrase)"

borg create "$REPO::$(date +%Y-%m-%d-%H%M%S)" \
  /home /etc /var/lib \
  --exclude-caches \
  --compression lz4 \
  && borg prune "$REPO" --keep-daily=7 --keep-weekly=4 \
  && curl -fsS --retry 3 https://hc-ping.com/YOUR-UUID > /dev/null
```

Both tools integrate cleanly with systemd timers, cron, and CI pipelines. The key practice is always pinging a monitoring endpoint on successful completion — silent backup failures are far more dangerous than noisy ones.

## Key Management: The Most Overlooked Risk

Encryption is only as durable as your key management. Both tools are excellent at encrypting data, but many developers lose access to encrypted backups not because of cryptographic failure but because of key loss.

**Restic key management** stores the repository master key in the repository itself, encrypted by your password. You can add multiple keys to a single repository, which is useful for team setups where multiple people need restore access:

```bash
# Add a second key (e.g., for a colleague)
restic -r /backup/restic-repo key add

# List all keys
restic -r /backup/restic-repo key list

# Remove an old key
restic -r /backup/restic-repo key remove <key-id>
```

The critical recovery document is your password. If you lose the password, the repository is unrecoverable — there is no vendor reset. Store your Restic password in a password manager, and print a copy stored in a physically secure location.

**Borg keyfile management** is more explicit. In `keyfile` mode, the key lives at `~/.config/borg/keys/`. This file is encrypted with your passphrase, but the file itself must be backed up separately — if you lose the key file and the passphrase, the archive is permanently inaccessible:

```bash
# Export the key for off-site backup
borg key export /backup/borg-repo /path/to/safe/borg-key.txt

# Import a key on a new machine
borg key import /backup/borg-repo /path/to/safe/borg-key.txt
```

A sound practice is to store the exported key file in your password manager as a secure note, and separately store the passphrase in a different location. This prevents a single point of failure from rendering your backups permanently unrecoverable.

In `repokey` mode (Borg's default), the encrypted key is embedded in the repository. This is more convenient but means a compromised repository server holds the encrypted key. For highly sensitive archives, `keyfile` mode is worth the additional management burden.

## Compression Options

Compression is a significant factor for storage efficiency, especially for text-heavy workloads like source code, logs, and database dumps.

| Tool | Compression Options | Best For |
|------|--------------------|-|
| Restic | zstd (default), lz4, gzip, bzip2 | General-purpose workloads |
| Borg | lz4 (fast), zstd, zlib, lzma | Tunable speed/ratio tradeoffs |

Borg's `--compression auto,lz4` mode automatically detects already-compressed data and skips compression for those chunks, saving CPU without reducing ratios for compressible data. Restic applies compression uniformly across chunks, which is simpler but slightly less efficient for mixed-content backups.

## Use Case Recommendations

Choose Restic when:

- Simplicity matters more than advanced features
- You need cross-platform support (Windows, macOS, Linux, BSD)
- Cloud storage backends (S3, B2, Azure, GCS) are your target
- You prefer a cleaner CLI with fewer flags

Choose Borg when:

- Local or network-attached storage is your primary target
- You need faster restore speeds
- Advanced verification features are important
- You have many similar machines to back up (deduplication shines)
- You want better compression ratios

Choose Restic when:

- Simplicity matters more than advanced features
- You need cross-platform support (Windows, macOS, Linux, BSD)
- Cloud storage backends (S3, B2, Azure, GCS) are your target
- You prefer a cleaner CLI with fewer flags

Choose Borg when:

- Local or network-attached storage is your primary target
- You need faster restore speeds
- Advanced verification features are important
- You have many similar machines to back up (deduplication shines)
- You want better compression ratios

## Security Considerations

Both tools provide strong encryption, but deployment matters:

1. **Never store passwords in shell history** — use environment variables or a secret manager
2. **Backup your repository keys** — Borg's keyfile or Restic's key must be preserved
3. **Test restores regularly** — encryption is useless if you cannot recover data
4. **Use key derivation parameters** — increase iterations/memory for sensitive data:

```bash
# Restic: custom key derivation
restic -r /backup/restic-repo unlock --max-size 100GB

# Borg: custom Argon2 parameters
borg init --encryption=keyfile-chacha20poly1305 \
    --argon2-memory=1048576 \
    --argon2-parallelism=4 \
    /backup/borg-repo
```

## Frequently Asked Questions

**Can I use the first tool and the second tool together?**

Yes, many users run both tools simultaneously. the first tool and the second tool serve different strengths, so combining them can cover more use cases than relying on either one alone. Start with whichever matches your most frequent task, then add the other when you hit its limits.

**Which is better for beginners, the first tool or the second tool?**

It depends on your background. the first tool tends to work well if you prefer a guided experience, while the second tool gives more control for users comfortable with configuration. Try the free tier or trial of each before committing to a paid plan.

**Is the first tool or the second tool more expensive?**

Pricing varies by tier and usage patterns. Both offer free or trial options to start. Check their current pricing pages for the latest plans, since AI tool pricing changes frequently. Factor in your actual usage volume when comparing costs.

**How often do the first tool and the second tool update their features?**

Both tools release updates regularly, often monthly or more frequently. Feature sets and capabilities change fast in this space. Check each tool's changelog or blog for the latest additions before making a decision based on any specific feature.

**What happens to my data when using the first tool or the second tool?**

Review each tool's privacy policy and terms of service carefully. Most AI tools process your input on their servers, and policies on data retention and training usage vary. If you work with sensitive or proprietary content, look for options to opt out of data collection or use enterprise tiers with stronger privacy guarantees.

## Related Articles

- [Best Encrypted Backup Solution For Developers](/privacy-tools-guide/best-encrypted-backup-solution-for-developers/)
- [Encrypted Backup Solutions Comparison 2026](/privacy-tools-guide/encrypted-backup-solutions-comparison-2026/)
- [Restic Encrypted Backup Setup Guide](/privacy-tools-guide/restic-encrypted-backup-setup-guide)
- [Privacy Focused Cloud Backup Services Comparison 2026](/privacy-tools-guide/privacy-focused-cloud-backup-services-comparison-2026/)
- [Encrypted Cloud Storage Comparison 2026: A Practical Guide](/privacy-tools-guide/encrypted-cloud-storage-comparison-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
