---
layout: default
title: "How to Audit Browser Extensions for Privacy Risks 2026"
description: "Audit Chrome and Firefox extensions for data exfiltration: CRXcavator scans, manifest.json analysis, and permission risk scoring methods."
date: 2026-03-22
last_modified_at: 2026-03-22
author: "Privacy Tools Guide"
permalink: /how-to-audit-browser-extensions-for-privacy-risks-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, browser-security, extension-audit, privacy]
---

{% raw %}

Browser extensions are powerful and dangerous. They run with high privileges in your browser, access your browsing history, can read everything you type, and contact external servers. A malicious or poorly-designed extension can steal your passwords, inject ads, monitor your activity, or sell your data.

The problem: most people install extensions without understanding the risks. You see an extension with millions of downloads and think it's safe. But downloads don't guarantee privacy. Facebook's official extension was collecting data; Microsoft's LinkedIn extension was doing the same.

This guide shows you how to audit extensions before installing them. You'll learn what permissions to question, how to read manifest files, how to detect suspicious behavior, and what tools reveal hidden data exfiltration.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1: Understand Browser Extension Permissions

When you install an extension, it asks for permissions. Most users click through without reading. This is a mistake.

Common permissions and what they actually mean:

"Access your data on all websites" - The extension can see every website you visit and everything on those websites. This is extremely powerful and necessary only for very few extensions. Password managers need this; most other extensions don't.

"Manage your tabs" - Read the URL of every tab you have open, and close or modify tabs. A tab manager needs this. A weather extension doesn't.

"Modify your download settings" - Intercept downloads, change download destinations, add download headers. A download manager needs this; most extensions don't.

"Access your browsing history" - See every website you've visited. This is powerful data. Only specialized history tools should need this.

"Read your email" - For Gmail extensions, read everything in your mailbox. Be careful with email integrations.

"Access camera/microphone" - Few extensions need this. If a productivity extension asks for camera access without explanation, question it.

"Make changes to search settings" - Modify your default search engine, search suggestions. Sketchy extensions often ask for this to inject advertising.

The key principle: an extension should ask for the minimum permissions to do its job. If an extension asks for more than necessary, that's a red flag. If you can't articulate why an extension needs a permission, don't grant it.

Step 2: Use CRXcavator to Analyze Extensions

CRXcavator is a free tool that analyzes Chrome extensions for suspicious behavior and risky permissions.

Go to crxcavator.io. Search for an extension by name. CRXcavator shows:

1. Risk score (0-100, lower is better): Based on permissions, version history, age, and update frequency.

2. Permissions analysis: Each permission is listed with severity level. See exactly what the extension can do.

3. Version history: How often is the extension updated? Frequent updates are good (security patches). No updates in 2 years is bad (abandoned, potentially vulnerable).

4. Content script hosts: What websites does the extension run on? An extension that runs on thousands of websites is more suspicious than one that runs on a few specific sites.

5. Network activity: What external domains does the extension contact? If an extension contacts data brokers, that's suspicious.

6. Manifest analysis: Details from the extension's manifest.json file, which defines what it can do.

A real example: searching for "Google Analytics" extensions on CRXcavator shows risk scores ranging from 30 (low risk) to 95 (very high risk). The high-risk ones request broad permissions like "access all websites" and have vague permission descriptions. The low-risk ones are limited to Google Analytics properties and clearly describe what they do.

Step 3: Reading manifest.json for Privacy Risks

The manifest.json file controls what an extension can do. You can view it even if you don't have the extension installed yet.

To find an extension's manifest without installing it:

1. Go to the Chrome Web Store
2. Find the extension's page
3. Click the extension ID in the URL (a long string of letters after /id/)
4. Look for the GitHub repository where the extension source is hosted (many open-source extensions publish on GitHub)

For open-source extensions, the manifest.json is directly viewable on GitHub.

Key manifest fields to check:

"permissions" array - Lists everything the extension is allowed to do. If it includes `<all_urls>` or broad patterns like `*://*/*`, the extension can run anywhere.

Example risky manifest:
```json
"permissions": [
  "<all_urls>",
  "webRequest",
  "cookies",
  "history"
],
```

This extension can see every website you visit, monitor your cookies, read your history, and intercept web requests. Only specialized tools like password managers should need this combination.

Example safer manifest:
```json
"permissions": [
  "https://mail.google.com/*",
  "storage"
],
```

This extension runs only on Gmail and stores local data. Much more limited.

"host_permissions" - Newer manifest format (Manifest V3) separates host permissions from other permissions. Similar principle: narrower is better.

"externally_connectable" - Specifies which external websites can send messages to the extension. Allows the extension to be remote-controlled by a website. If this lists untrustworthy domains, that's suspicious.

"content_scripts" - JavaScript code that runs on web pages. Check which sites it runs on and what it does. Some extensions inject ads or change page behavior here.

"background" - Background scripts run without an UI. They often contact external servers. Check if the manifest indicates what external services it contacts.

"update_url" - Where the extension gets updates. Most extensions use Google's update service. If this is a custom domain, be cautious (though some legitimate extensions do this).

Step 4: Detecting Data Exfiltration

How can you tell if an extension is sending your data to external servers? Browser developer tools help.

Method 1: Network Monitor

1. Open an extension's options page or activate its main feature
2. Open Developer Tools (right-click, Inspect)
3. Go to the Network tab
4. Look for external API calls

Watch what domains the extension contacts. If you see calls to analytics services, ad networks, or data brokers, that's data exfiltration.

Real example: A "productivity" extension claims to help you stay focused. When you activate it, the Network tab shows requests to:
- `api.analytics-company.com` - Analytics
- `ads.network.com` - Ad serving
- `tracking.databroker.com` - Data tracking

This extension is exfiltrating data. The developer is selling information about your productivity habits.

Method 2: Request Headers

In the Network tab, click a request. Look at the Request Headers section. Check for:

- Authorization tokens - If the extension sends an Authorization header with a token, that token can identify you and your activity.
- Cookie headers - The extension is sending cookies to external services (potential tracking).
- Custom headers with user identifiers - IDs that tie your data to your account across services.

If an extension sends your data to external services with identifying tokens, it's exfiltrating data.

Method 3: Storage Inspector

Open Developer Tools, go to Application > Storage. Check what the extension stores:

- Local Storage - Key-value data the extension stores locally
- Cookies - Any cookies set by the extension
- IndexedDB - More complex data storage

Look for suspicious stored data:
- User IDs associated with external services
- Tracking tokens
- Advertising IDs
- Browsing history copies

Legitimate extensions store minimal data. Sketchy extensions store copies of your browsing history, your location, or your search queries.

Step 5: Permissions Worth Questioning

"All websites" permission

Only password managers, full-page security tools, and privacy-focused adblockers legitimately need this. If a task-management or note-taking extension asks for it, question why. You can deny it and restrict the extension to specific sites using browser settings.

History access

Few extensions need this. A bookmark manager might. Most productivity tools don't. Deny it if the extension's features don't require it.

Tab management ("tabs" permission)

Tab managers need this. Productivity dashboards might. Ad-supported extensions sometimes ask for this to inject content into tabs. Check what the extension does with tab data.

Webrequest or webRequestBlocking

Allows the extension to modify web requests in flight. Legitimate uses: adblockers, security extensions, HTTPS enforcement. Sketchy uses: injecting tracking pixels, intercepting traffic, modifying responses.

Storage permissions

Most extensions need storage to save settings. This is low-risk. But check what they're storing (use the Storage Inspector).

Step 6: Red Flags in Extension Behavior

If you install an extension and notice:

1. Unexpected ads or search result hijacking - The extension is injecting ads or modifying search results. Data exfiltration is likely.

2. Slower browsing - The extension is doing something compute-intensive. Could be legitimate (adblocker, security tool) or malicious (mining crypto, tracking).

3. Changed homepage or search engine - The extension hijacked your browser settings. Uninstall immediately.

4. Strange permissions request after update - The extension updated and now asks for more permissions than before. The developers added a new feature that needs more access, or they're adding tracking.

5. Background activity when you're not using the extension - The Network tab shows requests to external services when the extension isn't actively doing anything. The extension is exfiltrating data in the background.

6. Downloaded to an unexpected location - Check your Downloads folder. Some extensions download additional binaries. This is extremely suspicious.

Step 7: Comparing Popular Extensions

Gmail Client Extensions

Gmail Offline (Google) - Made by Google. Runs only on mail.google.com. Stores encrypted local copies for offline access. Permissions: Minimal. Risk score: 25. Safe.

Simple Gmail Notes (third-party) - Runs on Gmail, adds notes capability. Permissions: Moderate (Gmail access, storage). Contacts: Notes to Google's servers for sync. Risk score: 45. Acceptable but monitor.

Gmail Tracker (third-party) - Claims to track email opens. Requests: All websites, webRequest, history. Contacts: Analytics servers, ad networks. Risk score: 92. Avoid.

Productivity Extensions

Todoist - Official client. Runs on multiple sites (Todoist plus email integration). Permissions: Moderate. Contacts: Todoist's own servers for sync. Risk score: 35. Safe.

Evernote Clipper - Official client. Captures web pages to Evernote. Permissions: Broad (all websites). Contacts: Evernote servers. Risk score: 50. Acceptable (necessary for capturing).

Honey (coupon addon) - Shows coupon codes while shopping. Requests: All websites, webRequest. Contacts: Honey's affiliate servers, PayPal. Risk score: 70. Privacy concern (tracks your shopping).

Step 8: Steps to Audit an Extension Before Installing

Step 1: Check CRXcavator

Search for the extension. If risk score > 70, reconsider installing.

Step 2: Read Recent Reviews

Go to the Chrome Web Store. Sort reviews by "Newest." Read them. If you see multiple complaints about ads, slowness, or unwanted behavior, avoid it.

Step 3: Check Update History

Look at the extension's update frequency. No updates in 1+ year? The extension may be abandoned and vulnerable. Frequent updates (weekly or monthly) are good.

Step 4: Review Permissions

Look at the permissions the extension asks for. Does each permission make sense for the extension's stated purpose? If you can't justify a permission, don't install it.

Step 5: Verify Developer Identity

Check who the developer is. Official extensions from Google, Microsoft, Slack, etc. are more trustworthy than fly-by-night developers.

Step 6: Search for Privacy Concerns

Google "[extension name] privacy" or "[extension name] data collection." See if there are known issues.

Step 7: Install and Monitor

If you decide to install, monitor it. Open Developer Tools once a week and check the Network tab. See what it's doing.

Step 8: Review Periodically

Every month, re-check CRXcavator for your installed extensions. Update frequency changes, new permissions might be requested, or the risk score might increase.

Step 9: What To Do If You Find a Risky Extension

If you discover an extension you installed is privacy-invasive:

1. Uninstall it - Don't use it, even if you've gotten used to its features.

2. Change passwords - If the extension had broad access, change passwords for sensitive accounts.

3. Clear browsing data - Clear cookies, cache, and site data to remove tracking.

4. Report it - Email the Chrome Web Store to report privacy concerns. If enough people report it, Google will review and potentially remove it.

5. Leave a review - Write a review on the Chrome Web Store warning others about the privacy issue.

6. Find a safer alternative - Most extensions have competitors. Use CRXcavator to find a safer option.

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How much can extensions see?

Depends on the permissions they have. With "all websites" permission, an extension can see every page you visit, everything you type, click, and select. They can also inject content into pages and modify page behavior.

Can I restrict extension permissions after installing?

Yes. Go to Settings > Extensions > Manage. Click the extension. Toggle "Allow on all sites" off and use "Allow on specified sites" to limit it to only the domains you need it on.

Are official extensions safer?

Usually, but not always. Official extensions are more trustworthy because the company's reputation is at stake. But even official extensions sometimes engage in questionable data practices. Always verify permissions.

Can extensions read my passwords?

Not directly, because browsers store passwords in encrypted form. But an extension with broad access can see password entry fields and intercept the data you type before it's encrypted. Use a password manager to mitigate this risk.

How often should I audit my extensions?

Audit monthly. Extensions update, permissions can change, and new risks emerge. CRXcavator scores change as Google reviews extensions.

What's the safest browser?

All modern browsers (Chrome, Firefox, Safari, Edge) are secure if you limit extension permissions. Edge and Safari are more restrictive by default. Firefox allows more control over permissions. Brave has excellent built-in privacy (adblocker, tracker blocking) so fewer extensions are needed.

Should I use an adblocker?

Yes, adblockers reduce your tracking exposure and improve performance. uBlock Origin (open-source, no data collection) is the best choice. Adblock Plus is acceptable but less strict about acceptable ads.

Related Articles

- [How to Audit Your Browser Extensions for Privacy](/how-to-audit-your-browser-extensions-for-privacy-risks/)
- [Privacy Implications of Browser Extensions](/privacy-implications-browser-extensions/)
- [How to Audit Chrome Extensions for Privacy](/audit-chrome-extensions-privacy-guide/)
- [Best Privacy Browser Extensions Ranked by Performance](/best-privacy-browser-extensions-ranked-by-performance-impact/)
- [Browser Extension Permissions What](/browser-extension-permissions-what-to-watch/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
