---
layout: default
title: "Macos Privacy Permissions Explained Which Tcc Database Entries To Revoke For Security"
description: "A guide for developers and power users on understanding macOS TCC database, identifying granted permissions, and revoking unnecessary."
date: 2026-03-16
author: theluckystrike
permalink: /macos-privacy-permissions-explained-which-tcc-database-entries-to-revoke-for-security/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

Inspect and revoke macOS TCC (Transparency, Consent, and Control) database entries to audit which apps have permission to access camera, microphone, location, contacts, and screen recording. Query the SQLite database directly with `sqlite3` commands to see granted permissions (auth_value=2), then delete entries for apps you no longer trust. This guide includes practical examples for revoking specific permissions and automating TCC audits across multiple machines.

## How TCC Works on macOS

Every time an application requests access to a protected resource—be it the camera, microphone, calendar, or desktop folder—macOS checks the TCC database. If no entry exists, the system prompts the user for permission. Once granted, the decision is stored in TCC, and subsequent requests are allowed automatically.

For developers, this means understanding TCC is critical when debugging permission-related issues. For power users, it means knowing which apps have been granted access helps identify potential privacy risks.

## Accessing the TCC Database

Reading the TCC database requires elevated privileges. Use the following commands to inspect granted permissions:

```bash
# Read system-wide TCC database (requires SIP disabled or full disk access)
sudo sqlite3 /Library/Application Support/com.apple.TCC/TCC.db "SELECT client, service, auth_value FROM access WHERE auth_value = 2;"

# Read user-level TCC database
sqlite3 ~/Library/Application Support/com.apple.TCC/TCC.db "SELECT client, service, auth_value FROM access WHERE auth_value = 2;"
```

The `auth_value` field indicates the permission level: `0` means denied, `1` means restricted, and `2` means allowed.

## Common TCC Services You Should Audit

Several TCC service identifiers control access to sensitive areas. Here are the most critical ones:

- `kTCCServiceCamera` — Camera access
- `kTCCServiceMicrophone` — Microphone access
- `kTCCServiceLocation` — Location services
- `kTCCServiceContacts` — Contacts database
- `kTCCServiceCalendar` — Calendar events
- `kTCCServiceReminders` — Reminders app
- `kTCCServicePhotos` — Photo library
- `kTCCServiceAccessibility` — Accessibility features (highly sensitive)
- `kTCCServiceScreenCapture` — Screen recording
- `kTCCServiceAppleMusic` — Apple Music access

## Revoking TCC Entries for Security

Removing TCC entries forces apps to re-request permission on next launch. This is useful when you want to reset permissions for an app you no longer trust, or to remove access for applications you've uninstalled.

### Method 1: Using tccutil (Limited)

Apple provides `tccutil` to reset permissions, but it only works for certain services and clears all entries:

```bash
# Reset all accessibility permissions
sudo tccutil reset Accessibility

# Reset all calendar permissions
tccutil reset Calendar
```

### Method 2: Direct SQLite Manipulation

For granular control, modify the TCC database directly:

```bash
# Backup the database first
cp ~/Library/Application Support/com.apple.TCC/TCC.db ~/TCC-backup.db

# Delete specific app permissions
sqlite3 ~/Library/Application Support/com.apple.TCC/TCC.db "DELETE FROM access WHERE client LIKE '%com.example.app%';"
```

For system-wide entries, use the sudo prefix and the system database path.

### Method 3: Using Third-Party Tools

Tools like **TCC Tracker** or **Privacy Generator** provide GUI interfaces for browsing and revoking TCC permissions. These are particularly useful for users uncomfortable with command-line operations.

## Practical Examples

### Example 1: Revoke Screen Recording from a Specific App

If you notice that a rarely-used utility has screen recording permission, revoke it:

```bash
sqlite3 ~/Library/Application Support/com.apple.TCC/TCC.db \
"DELETE FROM access WHERE service = 'kTCCServiceScreenCapture' AND client LIKE '%com.example.screenrecorder%';"
```

### Example 2: Check Which Apps Have Microphone Access

```bash
sqlite3 ~/Library/Application Support/com.apple.TCC/TCC.db \
"SELECT client FROM access WHERE service = 'kTCCServiceMicrophone' AND auth_value = 2;"
```

### Example 3: Reset All Permissions for an Uninstalled App

Even after removing an app, TCC entries may persist. Clean them up:

```bash
sqlite3 ~/Library/Application Support/com.apple.TCC/TCC.db \
"DELETE FROM access WHERE client LIKE '%com.uninstalled.app%';"
```

## Security Considerations

Revoking TCC entries enhances privacy but may break functionality for legitimate applications. Always verify which app is associated with a given bundle identifier before removal. You can find an app's bundle identifier using:

```bash
defaults read /Applications/AppName.app/Contents/Info CFBundleIdentifier
```

Additionally, some TCC entries are protected by System Integrity Protection (SIP). For these, you'll need to disable SIP temporarily or use recovery mode.

## Automating TCC Management

For developers managing multiple machines, script TCC audits:

```bash
#!/bin/bash
echo "=== TCC Permission Audit ==="
for db in ~/Library/Application\ Support/com.apple.TCC/TCC.db; do
  echo "Scanning: $db"
  sqlite3 "$db" "SELECT client, service FROM access WHERE auth_value = 2;" | \
  sort | uniq
done
```

Save this as `tcc-audit.sh` and run it periodically to track permission changes.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
