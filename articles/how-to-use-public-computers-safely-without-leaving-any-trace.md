---
layout: default
title: "How to Use Public Computers Safely Without Leaving Any Traces"
description: "A practical guide for developers and power users on using public computers securely. Learn browser fingerprinting prevention, data hygiene, and forensic countermeasures."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-public-computers-safely-without-leaving-any-trace/
categories: [security, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Public computers exist in libraries, hotels, cybercafes, and co-working spaces. Each session leaves behind data that can be recovered by the next user or forensic analysis. For developers and power users handling sensitive information, understanding these persistence mechanisms matters. This guide covers the technical reality of what gets left behind and practical countermeasures.

## What Stays Behind

Every action on a computer creates some form of persistent data. On public machines, this becomes a liability rather than a convenience.

**Browser data** represents the most obvious concern. Browsing history, cached files, cookies, and local storage persist across sessions. Even private or incognito modes have gaps—they do not prevent local caching of visited resources, and DNS resolution records often remain on the system.

**File system artifacts** extend beyond the browser. Recent documents lists, thumbnail caches, and file metadata (accessed via tools like `stat` on Linux or `Get-Item` on PowerShell) reveal activity. Applications like document editors auto-save drafts to hidden directories.

**Memory remnants** persist longer than expected. RAM contains decrypted session tokens, decrypted file contents, and keyboard input buffers. Cold boot attacks can recover these even after system shutdown, though this requires physical access and specialized hardware.

**Network artifacts** appear on both the client and server side. DHCP logs, ARP cache entries, and firewall connection tracking tables all record network activity. The public computer's network infrastructure may log DNS queries, connecting IP addresses, and traffic patterns.

## Browser Hardening

The browser is your primary interface and your biggest vulnerability. Several hardening measures reduce your fingerprint.

### Portable Browsers

Portable browser installations run from USB drives without installing to the local system. They store all data—including cookies, history, and extensions—within their own directory structure. The tradeoff involves decreased performance (USB 2.0 bandwidth limitations) and the need to trust the machine's operating system not to inject keyloggers or memory scrapers.

```bash
# Verify browser integrity after download (example for Firefox)
sha256sum firefox-setup.tar.bz2
gpg --verify firefox-setup.tar.bz2.asc
```

### Browser Configuration

Disable features that persist across sessions. These settings live in `about:config` for Firefox or Chromium-based browsers:

```
# Firefox settings to disable
privacy.resistFingerprinting = true
network.cookie.cookieBehavior = 1  # Block third-party cookies
browser.sessionstore.max_tabs_undo = 0
browser.cache.disk.enable = false
browser.cache.memory.enable = false
```

For Chromium-based browsers, launch with command-line switches:

```bash
chromium --incognito \
  --disable-background-networking \
  --disable-default-apps \
  --disable-extensions \
  --disable-sync \
  --disable-translate \
  --enable-features=NetworkService,NetworkServiceInProcess \
  --disk-cache-size=0 \
  --media-cache-size=0 \
  --disable-features=TranslateUI \
  "https://example.com"
```

### Cookie and Session Management

Precious few websites function without sessions. When you must authenticate on a public machine, consider these approaches:

1. **Browser-based session tokens**: After login, inspect `document.cookie` and clear it via developer tools before closing. Note that many sites set multiple cookies across different domains.

2. **Token expiration**: Use services that issue short-lived tokens. OAuth2 tokens from GitHub expire after eight hours by default, but you can request shorter-lived tokens via the token scope parameters.

3. **Dedicated authentication devices**: Hardware security keys (YubiKey, Titan) store credentials in isolated hardware. The private key never leaves the device, and the public computer never sees your actual credentials—only a cryptographic assertion.

## Network-Level Countermeasures

Your network traffic reveals significant information even when browser hardening works.

### VPN Usage

A trusted VPN encrypts traffic between your device and the exit node, preventing the public network from observing your destination addresses. However, this only works if you control the VPN client configuration—public computers cannot be trusted to run trusted VPN software.

The solution involves carrying your own:

```bash
# WireGuard client configuration (client.conf)
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = vpn.example.com:51820
PersistentKeepalive = 25
```

Deploy this configuration from your own USB drive. Verify the VPN functions by checking `ifconfig` or `ip addr` to confirm the tunnel interface exists.

### DNS Considerations

DNS queries reveal your browsing destinations. Even with HTTPS, DNS resolution happens in plaintext (unless using DNS-over-HTTPS or DNS-over-TLS). On a public machine, you cannot configure the system's DNS resolver without administrator privileges.

For sensitive sessions, carry a SOCKS5 or HTTP proxy on your USB and tunnel all traffic through it:

```bash
# SSH-based SOCKS proxy
ssh -D 1080 -N -f user@your-server.example

# Configure browser to use localhost:1080 as SOCKS5 proxy
```

## File System and OS Hygiene

Beyond the browser, your session creates additional artifacts.

### Temporary File Management

Operating systems create temporary files extensively. Modern Windows systems use `C:\Users\<username>\AppData\Local\Temp`, while Linux systems typically use `/tmp` and `/var/tmp`.

Some applications respect the `TMPDIR` environment variable:

```bash
# Set temp directory to RAM disk (Linux)
export TMPDIR=/dev/shm
export TEMP=/dev/shm
export TMP=/dev/shm

# Create RAM disk if not present
sudo mkdir -p /dev/shm
sudo chmod 1777 /dev/shm
```

The `/dev/shm` directory exists in RAM on most Linux systems, meaning files disappear on reboot or power loss.

### Recent Files and Jump Lists

Windows stores recent files in multiple locations: jump lists, recent documents folders, and the file explorer's recent view. Disable these via group policy or registry:

```powershell
# PowerShell - Disable recent files
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" -Name "NoRecentDocsHistory" -Value 1
Remove-Item "$env:APPDATA\Microsoft\Windows\Recent\*" -Force -ErrorAction SilentlyContinue
```

On Linux, check and clear `~/.local/share/recently-used.xbel` and similar files.

## Memory and Cold Boot Considerations

Data in RAM presents an advanced threat model but remains relevant for high-security scenarios.

### Secure Memory Wipe

Before leaving a public machine, you cannot reliably wipe RAM without special software. However, you can minimize the window of exposure by:

1. Closing all applications before walking away
2. Suspending the system (if safe) rather than shutting down, as cold boot attacks target shutdown states
3. Avoiding hibernate modes that write RAM to disk

For Linux systems with `memfd` support, you can create and immediately destroy sensitive data in memory:

```c
// Secure memory wipe example (C)
#include <stdlib.h>
#include <string.h>

void secure_wipe(void *ptr, size_t size) {
    memset(ptr, 0, size);
    __asm__ __volatile__("" ::: "memory");
    free(ptr);
}
```

### Swap Considerations

Linux systems may write sensitive memory pages to swap. Disable swap or encrypt it:

```bash
# Check current swap usage
swapon --show

# Disable all swap (requires sufficient RAM)
swapoff -a
```

## Alternative: Never Authenticate on Public Machines

The most secure approach avoids the problem entirely. Use these strategies instead:

- **Pre-session preparation**: Complete all authentication on your trusted device before arriving at the public computer. Copy necessary data to encrypted USB if needed.

- **Read-only access**: Access public-facing resources that do not require authentication. Use cached content from your trusted device.

- **Two-factor authentication forward**: If you must authenticate, use a password manager's one-time password (OTP) generation on your mobile device rather than the public computer's browser.

- **Delegate to your phone**: Some services offer mobile-friendly interfaces. Authenticate on your phone, then use bluetooth or local network features to relay the session to the public computer.

## Quick Checklist

Before using a public computer for any sensitive activity:

- [ ] Use portable browser from trusted USB
- [ ] Disable browser caching and history features
- [ ] Connect through your own VPN or SOCKS proxy
- [ ] Set `TMPDIR` to RAM disk if available
- [ ] Clear recent files and browser data after session
- [ ] Close all applications before leaving
- [ ] Prefer not to authenticate at all when possible

Public computers will always carry residual risk. Understanding what data persists and applying defense-in-depth measures reduces your attack surface significantly. For developers handling API keys, credentials, or proprietary code, these practices prevent accidental exposure that could compromise production systems.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
