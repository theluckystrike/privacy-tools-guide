---
layout: default
title: "Nextcloud External Storage Setup Guide 2026"
description: "A technical guide to configuring external storage in Nextcloud for developers and power users. Connect S3, WebDAV, FTP, and local storage"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /nextcloud-external-storage-setup-guide-2026/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
Nextcloud's external storage app lets you mount S3 buckets, WebDAV servers, FTP sites, and local directories directly into your Nextcloud file browser, creating an unified interface for all your data regardless of where it physically lives. Enable the External Storage support app in the admin interface, then create mount points by specifying storage type, credentials, and mount path. This guide covers configuration for all major backends with practical examples for developers managing multiple storage sources.

## Prerequisites

Before configuring external storage, ensure you have:

- A working Nextcloud installation (version 27 or later recommended)
- The External Storage support app enabled in your Nextcloud admin settings
- Appropriate credentials or access keys for your chosen storage backend
- Sufficient permissions to create mount points in Nextcloud

To verify the external storage app is enabled, navigate to **Apps > Your apps** in the Nextcloud admin interface and confirm "External storage support" is active.

## Adding External Storage via the Web Interface

The simplest method uses Nextcloud's web interface. Log in as an administrator and navigate to **Settings > Administration > External storage**.

### Adding Local Storage

For mounting directories outside the default Nextcloud data folder:

1. In the External storage section, select **Local** from the storage dropdown
2. Enter a mount point name (e.g., `/media/backup-drive`)
3. Configure permissions — the web server user needs read/write access:

```bash
# Ensure Nextcloud can access the directory
sudo chown -R www-data:www-data /mnt/shared-data
sudo chmod 755 /mnt/shared-data
```

4. Set the folder name that will appear in Nextcloud
5. Click the checkmark to save

The mounted directory now appears in Nextcloud's file browser alongside your main storage.

### Configuring S3-Compatible Storage

For AWS S3, MinIO, Backblaze B2, or other S3-compatible providers:

1. Select **Amazon S3** from the storage dropdown
2. Enter your credentials:
 - Bucket name
 - Access key ID
 - Secret access key
 - Region (or leave blank for MinIO)
3. Configure the hostname for MinIO or other S3-compatible services:

```bash
# MinIO configuration example
Hostname: minio.example.com
Port: 9000
```

4. Enable "SSL" for production deployments
5. Save the configuration

The S3 bucket mounts as a folder in Nextcloud, with files stored remotely but accessible through the web interface. Files are not automatically downloaded — Nextcloud accesses them on-demand unless you enable caching.

### WebDAV Mounts

For connecting to ownCloud, Nextcloud instances, or WebDAV-enabled servers:

1. Select **WebDAV** from the storage dropdown
2. Enter the WebDAV URL:

```bash
# WebDAV URL format
https://remote-server.com/remote.php/dav/files/username/
```

3. Provide credentials or use an app password
4. Configure the remote subfolder if needed

WebDAV mounts support both basic authentication and OAuth2, depending on your server configuration.

## Command-Line Configuration

For automated deployments or scripted setups, use the Nextcloud `occ` command:

```bash
# List current external storage mounts
occ files_external:list

# Create an S3 mount
occ files_external:amazon-s3 \
  --bucket=my-bucket \
  --hostname=s3.amazonaws.com \
  --region=us-east-1 \
  --access-key=AKIAIOSFODNN7EXAMPLE \
  --secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  --mount-point=s3-archive \
  --user=admin

# Create a local mount
occ files_external:local \
  --path=/mnt/external-drive \
  --mount-point=archive \
  --user=admin
```

The `occ files_external:applicable` command controls which users or groups can access specific mounts.

## Programmatic Access with the WebDAV API

Once external storage is mounted, access files programmatically using Nextcloud's WebDAV endpoint:

```python
import requests
from requests.auth import HTTPBasicAuth

base_url = "https://cloud.example.com/remote.php/dav/files/admin/"
auth = HTTPBasicAuth("admin", "your-app-password")

# List files in external storage mount
response = requests.request(
    "PROPFIND",
    base_url + "s3-archive/",
    auth=auth,
    headers={"Depth": "1"}
)

print(response.text)

# Upload a file
with open("backup.tar.gz", "rb") as f:
    response = requests.put(
        base_url + "s3-archive/backup.tar.gz",
        auth=auth,
        data=f,
        headers={"Content-Type": "application/gzip"}
    )
```

This approach works with any external storage backend — Nextcloud handles the translation between its WebDAV interface and the underlying storage type.

## Performance Considerations

External storage introduces latency compared to local Nextcloud storage. Optimize performance with these strategies:

**Enable caching** in your mount configuration to store frequently accessed files locally:

```bash
# Via occ command
occ files_external:option 1 enable_cache true
```

**Configure preload** for directories with predictable access patterns:

```bash
occ files_external:option 1 preallocate 100
```

**Use the filesystem scanner** sparingly — for large S3 buckets, consider using S3-native tools for bulk operations and only mount specific prefixes in Nextcloud.

## Security Best Practices

When configuring external storage, follow these security practices:

- Use IAM roles or limited-access credentials instead of root AWS keys
- Enable TLS/SSL for all network-based storage backends
- Restrict mount access to specific users or groups via the permissions interface
- Regularly audit external storage configurations using `occ files_external:list`
- Store credentials in Nextcloud's secure credential storage rather than plain configuration

## Troubleshooting Common Issues

**Permission denied errors**: Verify the web server user has filesystem access to local mounts. Check SELinux or AppArmor policies on Linux systems.

**S3 authentication failures**: Confirm credentials are correct and the IAM user has s3:GetObject and s3:PutObject permissions for the specific bucket.

**Slow performance**: Enable caching, check network latency to the storage backend, and consider mounting only necessary subdirectories rather than entire buckets.

**Mount not appearing**: Clear the Nextcloud cache and verify the mount configuration is saved:

```bash
occ files_external:verify 1
occ maintenance:repair
```

## Use Cases for Developers

External storage excels in several developer-focused scenarios:

- **Artifact storage**: Connect S3 buckets to store build artifacts, release packages, and deployment scripts
- **Multi-instance sharing**: Mount a shared WebDAV folder across multiple Nextcloud instances for team collaboration
- **Backup consolidation**: Aggregate backups from multiple servers into a single S3-backed archive
- **Media libraries**: Connect large media storage without consuming local disk space



## Related Articles

- [Self Hosted Cloud Storage Comparison Nextcloud vs.](/privacy-tools-guide/self-hosted-cloud-storage-comparison-nextcloud-vs-seafile-vs-syncthing/)
- [Nextcloud Collabora Office Setup Guide](/privacy-tools-guide/nextcloud-collabora-office-setup-guide/)
- [Nextcloud End to End Encryption Setup Guide](/privacy-tools-guide/nextcloud-end-to-end-encryption-setup-guide/)
- [Nextcloud Setup Guide Raspberry Pi 2026](/privacy-tools-guide/nextcloud-setup-guide-raspberry-pi-2026/)
- [Nextcloud Talk Video Calls Setup Guide](/privacy-tools-guide/nextcloud-talk-video-calls-setup-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
