---
layout: default
title: "Restic vs Borg Backup: Encrypted Comparison for Developers"
description: "A technical comparison of Restic and Borg backup tools focusing on encryption, deduplication, and performance for developers and power users."
date: 2026-03-15
author: theluckystrike
permalink: /restic-vs-borg-backup-encrypted-comparison/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

When selecting a backup solution for sensitive data, encryption becomes the deciding factor. Restic and Borg are both open-source, deduplicating backup programs, but their encryption approaches differ significantly. This comparison examines how each tool handles encryption, performance, and usability from a developer's perspective.

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

## Conclusion

For encrypted backups, both Restic and Borg deliver solid security with strong deduplication. Restic offers simpler cloud storage integration and a cleaner CLI, while Borg provides faster restores and more advanced repository management. Evaluate based on your storage backend, restore frequency, and whether you need cross-platform consistency. Test both with your actual data before committing to production use.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
