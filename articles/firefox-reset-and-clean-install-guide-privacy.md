---

layout: default
title: "Firefox Reset And Clean Install Guide Privacy"
description: "Complete guide to resetting and performing a clean install of Firefox for maximum privacy. Learn backup strategies, profile management, and."
date: 2026-03-15
author: theluckystrike
permalink: /firefox-reset-and-clean-install-guide-privacy/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

To reset Firefox while keeping your data, go to `about:support` and click "Refresh Firefox" -- this preserves bookmarks and passwords but removes extensions and resets `about:config`. For a full clean install that eliminates all accumulated tracking data, back up your profile first, then delete your profile directory and reinstall. Below you will find step-by-step instructions for both methods, including backup commands, privacy hardening settings to apply immediately after reset, and an automation script for power users.

## When to Reset or Clean Install Firefox

Consider resetting Firefox when you notice performance degradation, encounter persistent extension conflicts, or want to start with a clean privacy baseline. A clean install differs from a reset—the former removes all profile data completely, while the latter attempts to preserve bookmarks, passwords, and preferences.

Developers and power users benefit from periodic clean installs because browser state accumulates residual tracking identifiers over time. Even privacy-conscious users accumulate cookies, localStorage entries, and browser fingerprinting data through normal usage.

## Backing Up Essential Data

Before performing any reset or clean install, export critical data. Firefox stores profiles in platform-specific directories:

- **Linux**: `~/.mozilla/firefox/`
- **macOS**: `~/Library/Application Support/Firefox/Profiles/`
- **Windows**: `%APPDATA%\Mozilla\Firefox\Profiles\`

Each profile folder contains a randomly generated identifier. Locate your active profile by checking `profiles.ini` in the Firefox directory, or simply open `about:support` in Firefox and click "Open Directory" next to "Profile Folder."

### Exporting Bookmarks

```bash
# Using Firefox's places.sqlite
# This requires sqlite3 installed
sqlite3 ~/.mozilla/firefox/*.default-release/places.sqlite \
  "SELECT p.title, b.url FROM moz_places b 
   JOIN moz_bookmarks p ON b.id = p.fk 
   ORDER BY p.parent, p.position;" > bookmarks_export.html
```

For a more reliable approach, use Firefox's built-in JSON backup:

1. Open Bookmarks Manager (Ctrl+Shift+O)
2. Click "Import and Backup" → "Backup..."
3. Save as JSON for maximum data fidelity

### Exporting Passwords

```bash
# Export passwords using Python and the logins.json file
# Requires Python 3.x with sqlite3 (usually included)
python3 << 'EOF'
import json
import os
import sys

profile_path = os.path.expanduser("~/.mozilla/firefox/")
for root, dirs, files in os.walk(profile_path):
    if "logins.json" in files:
        with open(os.path.join(root, "logins.json")) as f:
            data = json.load(f)
            for entry in data["logins"]:
                print(f"{entry['hostname']}:{entry['username']}")
        break
EOF
```

**Security note**: These exported files contain sensitive data. Encrypt them before storing or transfer them to a secure password manager.

## Method 1: Firefox Reset (Built-in Refresh)

Firefox's built-in reset feature refreshes the browser while attempting to preserve essential data:

1. Navigate to `about:support`
2. Click "Refresh Firefox" button
3. Confirm the action

This process:
- Creates a new profile
- Copies bookmarks, history, cookies, and saved passwords to the new profile
- Disables all extensions
- Resets `about:config` preferences to defaults
- Removes custom search engines (except defaults)

The reset preserves your sync account data, so bookmarks and preferences sync back after signing in again. However, this method may retain some tracking data embedded in site permissions and exceptions.

## Method 2: Clean Install (Complete Profile Removal)

A true clean install removes all profile data. This provides the strongest privacy baseline but requires manual data restoration.

### Step 1: Completely Close Firefox

Ensure Firefox isn't running in the background:

```bash
# Linux/macOS - kill all Firefox processes
pkill -f firefox

# Verify no processes remain
pgrep -f firefox && echo "Firefox still running" || echo "Firefox stopped"
```

### Step 2: Remove Firefox Profile Data

```bash
# Navigate to Firefox profile directory
cd ~/.mozilla/firefox/

# List all profiles to identify the one to remove
cat profiles.ini

# Remove the entire profile directory (replace with your actual profile folder)
rm -rf *.default-release
```

**Warning**: This permanently deletes all data in that profile. Ensure backups exist first.

### Step 3: Reinstall Firefox

For privacy-sensitive usage, consider Firefox's specialized builds:

- **Firefox Standard**: Standard release with privacy features
- **Firefox Extended Support Release (ESR)**: Stable, long-term support version
- **Firefox Nightly**: Latest features with experimental privacy protections

Download from the official Mozilla repository:

```bash
# Example: Installing Firefox ESR on Ubuntu/Debian
sudo apt update
sudo apt install firefox-esr
```

### Step 4: Post-Install Privacy Configuration

After clean install, apply privacy hardening immediately before browsing:

```javascript
// Paste these into about:config (boolean = true, integer values as shown)

// Enhanced tracking protection - strict mode
privacy.trackingprotection.enabled = true
privacy.trackingprotection.stricttracking.enabled = true

// Resist fingerprinting
privacy.resistFingerprinting = true

// Disable telemetry
datareporting.policy.dataSubmissionEnabled = false
browser.newtabpage.enabled = false

// Disable WebGL (reduces fingerprinting surface)
webgl.disabled = true

// Clear on exit
privacy.clearOnShutdown.cache = true
privacy.clearOnShutdown.cookies = true
privacy.clearOnShutdown.downloads = true
privacy.clearOnShutdown.formdata = true
privacy.clearOnShutdown.history = true
privacy.clearOnShutdown.offlineApps = true
privacy.clearOnShutdown.sessions = true

// Set third-party cookie blocking
network.cookie.cookieBehavior = 1  // Block third-party cookies

// Disable autofill for sensitive data
signon.autofillForms = false
```

Apply these preferences carefully—some settings may break functionality on certain websites.

## Verifying Privacy Hardening

After configuration, verify your privacy posture:

1. **Check browser fingerprint**: Visitcovery.jp or amiunique.org to see how identifiable your browser appears
2. **Test tracking protection**: Visit some known tracker-heavy sites and check the shield icon in the address bar
3. **Verify no Telemetry**: Navigate to `about:telemetry`—it should show "Telemetry disabled"

## Restoring Data Selectively

Rather than restoring everything, selectively import only essential data:

1. Import bookmarks from the JSON backup
2. Manually re-enter critical passwords (forces you to audit stored credentials)
3. Reinstall extensions one by one, evaluating privacy implications each time

This selective restoration prevents transferring old tracking data to your fresh installation.

## Automation for Power Users

Developers can automate Firefox privacy hardening using a startup script:

```bash
#!/bin/bash
# firefox-privacy-setup.sh

PROFILE_DIR=$(find ~/.mozilla/firefox -maxdepth 1 -type d -name "*.default*" | head -1)

if [ -z "$PROFILE_DIR" ]; then
    echo "No Firefox profile found"
    exit 1
fi

# Create user.js for persistent privacy settings
cat > "$PROFILE_DIR/user.js" << 'EOF'
// Privacy hardening settings
user_pref("privacy.trackingprotection.enabled", true);
user_pref("privacy.resistFingerprinting", true);
user_pref("webgl.disabled", true);
user_pref("network.cookie.cookieBehavior", 1);
user_pref("datareporting.policy.dataSubmissionEnabled", false);
user_pref("browser.newtabpage.enabled", false);
EOF

echo "Privacy settings applied to $PROFILE_DIR"
```

Run this script after each Firefox update, as some settings may reset.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [Firefox Arkenfox User.js Full Guide: Complete Privacy.](/privacy-tools-guide/firefox-arkenfox-user-js-full-guide/)
- [Firefox Privacy Add-ons Essential List 2026: Complete Guide](/privacy-tools-guide/firefox-privacy-add-ons-essential-list-2026/)
- [Firefox Privacy Settings Guide 2026: A Practical Guide.](/privacy-tools-guide/firefox-privacy-settings-guide-2026/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
