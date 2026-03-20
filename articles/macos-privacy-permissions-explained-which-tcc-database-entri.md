---
layout: default
title: "Macos Privacy Permissions Explained Which Tcc Database."
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

## Extended TCC Service Reference

Beyond the common services, macOS includes many additional TCC entries that control privacy-sensitive access. Here's a comprehensive reference:

| Service ID | Description | Risk Level |
|------------|-------------|-----------|
| `kTCCServiceCamera` | Camera access | High |
| `kTCCServiceMicrophone` | Microphone access | High |
| `kTCCServiceScreenCapture` | Screen recording | High |
| `kTCCServiceAccessibility` | Accessibility (keyboard, mouse tracking) | Critical |
| `kTCCServiceLocation` | Location services | High |
| `kTCCServiceContacts` | Contacts database | High |
| `kTCCServiceCalendar` | Calendar events | Medium |
| `kTCCServiceReminders` | Reminders app | Medium |
| `kTCCServicePhotos` | Photo library | High |
| `kTCCServiceMediaLibrary` | Music and video library | Medium |
| `kTCCServiceWillow` | HomeKit device access | Medium |
| `kTCCServiceTCC` | TCC database itself | Critical |
| `kTCCServiceSystemPolicyDocumentsFolder` | Documents folder access | High |
| `kTCCServiceSystemPolicyDownloadsFolder` | Downloads folder access | High |
| `kTCCServiceSystemPolicyDesktopFolder` | Desktop folder access | High |

## Advanced TCC Queries

For detailed permission audits, use more sophisticated SQLite queries:

```bash
# See full TCC database schema
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db ".schema"

# Export all permissions with metadata
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, auth_value, last_modified FROM access ORDER BY last_modified DESC;" \
  | column -t -s '|'

# Find apps with multiple sensitive permissions
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, COUNT(*) as perm_count FROM access
   WHERE auth_value = 2 AND service IN
   ('kTCCServiceCamera', 'kTCCServiceMicrophone', 'kTCCServiceScreenCapture', 'kTCCServiceAccessibility')
   GROUP BY client HAVING perm_count > 1;" \
  | column -t -s '|'

# Track when permissions were granted (requires full_disk_access for system DB)
sudo sqlite3 /Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, auth_value, datetime(last_modified, 'unixepoch')
   FROM access WHERE auth_value = 2 ORDER BY last_modified DESC LIMIT 20;"
```

## System Integrity Protection and TCC

Apple's System Integrity Protection (SIP) prevents modification of certain system-level TCC entries even with sudo. Entries for system processes and critical services are locked:

```bash
# Check SIP status
csrutil status

# To modify system-level TCC entries, you must disable SIP
# 1. Restart into Recovery Mode (Cmd+R during startup)
# 2. Open Terminal from Utilities
# 3. Run: csrutil disable
# 4. Restart your Mac
# 5. Modify TCC entries as needed
# 6. Re-enable SIP: sudo csrutil enable
# 7. Restart again
```

Be cautious with SIP modifications—disabling it reduces system security. Only disable for legitimate maintenance and re-enable immediately after.

## TCC Database Corruption and Recovery

If TCC operations fail, the database may be corrupted or locked:

```bash
# Check file permissions on TCC database
ls -la ~/Library/Application\ Support/com.apple.TCC/TCC.db

# Verify database integrity
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db "PRAGMA integrity_check;"

# Rebuild corrupted database
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db ".dump" > tcc_backup.sql
# Then delete the corrupted DB and restore from backup after fixing

# Check if any processes have the TCC database locked
lsof ~/Library/Application\ Support/com.apple.TCC/TCC.db
```

If the TCC database is locked by a running process, kill that process or restart your Mac to release the lock.

## Automating TCC Management for Security Teams

For developers managing multiple machines, create comprehensive TCC audit and deployment scripts:

```bash
#!/bin/bash
# Comprehensive TCC Permission Audit Script

echo "=== TCC Permission Audit Report ==="
echo "Generated: $(date)"
echo "Hostname: $(hostname)"
echo ""

# Function to audit TCC database
audit_tcc_db() {
    local db_path=$1
    local db_type=$2

    if [ ! -f "$db_path" ]; then
        echo "Database not found: $db_path"
        return
    fi

    echo "=== $db_type Permissions ==="
    sqlite3 "$db_path" \
      "SELECT client, service,
              CASE auth_value WHEN 0 THEN 'Denied' WHEN 1 THEN 'Restricted' WHEN 2 THEN 'Allowed' END as status
       FROM access
       WHERE auth_value = 2
       ORDER BY client, service;" | \
      awk -F'|' '{printf "%-50s %-35s %s\n", $1, $2, $3}'
    echo ""
}

# Audit user-level database
audit_tcc_db ~/Library/Application\ Support/com.apple.TCC/TCC.db "User-Level"

# Audit system-level database (requires Full Disk Access)
if [ -f "/Library/Application Support/com.apple.TCC/TCC.db" ]; then
    audit_tcc_db /Library/Application\ Support/com.apple.TCC/TCC.db "System-Level"
fi

# Export results for tracking
echo ""
echo "=== High-Risk Permissions Detected ==="
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client FROM access
   WHERE auth_value = 2 AND service IN
   ('kTCCServiceAccessibility', 'kTCCServiceScreenCapture', 'kTCCServiceCamera');"
```

Save as `tcc-audit.sh`, make executable, and run periodically to track permission changes and identify new high-risk grants.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
