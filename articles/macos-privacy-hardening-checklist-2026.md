---
layout: default
title: "macOS Privacy Hardening Checklist 2026"
description: "A complete macOS privacy checklist covering Gatekeeper, Siri, telemetry, location services, firewall, iCloud, and system-level settings for Sequoia and ..."
date: 2026-03-21
author: theluckystrike
permalink: /macos-privacy-hardening-checklist-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

macOS collects more data than most users realize. Apple telemetry, Siri voice processing, iCloud photo analysis, location history, and app analytics all transmit data off your device, even when you haven't explicitly opted in to any of them. This checklist covers every privacy-relevant setting in macOS Sequoia (15.x) and applies broadly to macOS 13 and 14 as well.

Work through each section. Many of these settings are buried in submenus that most people never see.

Table of Contents

- [System Settings: Privacy and Security](#system-settings-privacy-and-security)
- [Network Privacy Settings](#network-privacy-settings)
- [iCloud Settings](#icloud-settings)
- [Software and Updates](#software-and-updates)
- [Safari Privacy Settings](#safari-privacy-settings)
- [Terminal Hardening Commands](#terminal-hardening-commands)
- [FileVault](#filevault)
- [Audit Installed Apps and Login Items](#audit-installed-apps-and-login-items)
- [Related Reading](#related-reading)

System Settings: Privacy and Security

Open System Settings → Privacy & Security and work through each category:

Location Services

- Set to Off for any app that does not have a clear reason to need it
- Review apps with "Always" access. this means they can locate you in the background
- Disable location access for: Weather (use manual location), Spotlight Suggestions, Siri
- Under System Services → Details: disable "Significant Locations," "iPhone Analytics," "Mac Analytics," "Routing and Traffic," "Improve Maps"

```bash
Disable Location Services entirely via command line (requires SIP disabled or MDM)
sudo defaults write /var/db/locationd/Library/Preferences/ByHost/com.apple.locationd LocationServicesEnabled -int 0
```

Analytics and Improvements

Uncheck all of:
- Share Mac Analytics
- Improve Siri and Dictation
- Share with App Developers
- Share iCloud Analytics

Apple Advertising

- Disable "Personalized Ads"

Siri and Spotlight

System Settings → Siri and Spotlight:
- Disable "Ask Siri"
- Disable "Listen for 'Hey Siri'"
- Under Siri Suggestions, disable all categories
- Disable "Improve Siri and Dictation" (also in Privacy → Analytics)

For Spotlight, disable online search suggestions:
- System Settings → Siri and Spotlight → Search Results → uncheck "Allow Spotlight Suggestions"

App Permissions Audit

In Privacy & Security, review each category:
- Camera and Microphone: Remove any app you don't recognize or don't actively use for video/audio
- Contacts, Calendars, Reminders: Limit to apps that genuinely need sync
- Full Disk Access: Remove anything unexpected here. this grants access to all files
- Accessibility: High-risk permission; every app listed can control your entire Mac

Network Privacy Settings

iCloud Private Relay

If you have an iCloud+ subscription, enable iCloud Private Relay: System Settings → Apple ID → iCloud → Private Relay.

Private Relay routes Safari traffic through two separate internet relay servers. Apple sees your IP but not your destination, the relay operator sees the destination but not your IP. It only applies to Safari and does not protect other apps.

DNS Configuration

Replace your ISP's default DNS with a privacy-respecting resolver:

```bash
Set DNS via networksetup (replace "Wi-Fi" with your interface name)
networksetup -setdnsservers Wi-Fi 9.9.9.9 149.112.112.112

Verify
scutil --dns | grep nameserver
```

Consider Quad9 (9.9.9.9), NextDNS, or Mullvad DNS. All offer DNS-over-HTTPS or DNS-over-TLS.

Built-in Firewall

System Settings → Network → Firewall:
- Enable the firewall
- Click Options → check "Block all incoming connections" (note: this blocks file sharing, screen sharing, AirDrop)
- Enable "Stealth Mode". prevents the Mac from responding to ICMP probes

```bash
Enable firewall via command line
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
```

iCloud Settings

iCloud syncs a large amount of sensitive data. Review what is synced:

System Settings → Apple ID → iCloud → Show All:

- iCloud Drive: Off if you don't need cloud sync
- Photos: Off unless you want Apple servers to store your photos (even with E2E encryption, metadata may be accessible)
- Contacts/Calendars: Consider local-only or self-hosted alternatives
- iCloud Backup (iPhone sync): Stores your device backups. ensure Advanced Data Protection is enabled if you use this
- Siri: Turn off "Sync Siri across devices"

Advanced Data Protection

If you use iCloud, enable Advanced Data Protection: Apple ID → iCloud → Advanced Data Protection. This expands end-to-end encryption to cover iCloud Backup, Photos, Notes, and most other iCloud data. Apple Support cannot read this data even with a legal request.

Software and Updates

Automatic Updates

System Settings → General → Software Update → Automatic Updates:
- Enable all automatic updates. Prompt: "won't this phone home?". yes, but running outdated software with known vulnerabilities is a worse trade-off.

Gatekeeper and Notarization

Keep Gatekeeper on. System Settings → Privacy & Security → Security:
- Set to "App Store and identified developers" at minimum
- Do not set to "Anywhere". this disables Gatekeeper entirely

To run a specific unsigned app without disabling Gatekeeper:
```bash
sudo spctl --add /Applications/AppName.app
```

Safari Privacy Settings

If you use Safari rather than a privacy-hardened browser:

Safari → Settings → Privacy:
- Enable "Prevent cross-site tracking"
- Enable "Hide IP address from trackers"
- Check "Block all cookies" only if you can tolerate site breakage

Safari → Settings → Search:
- Change Search Engine from Google to DuckDuckGo or Brave Search
- Disable "Include search engine suggestions"
- Disable "Preload Top Hit in Background"

Terminal Hardening Commands

Run these commands to tighten privacy settings that have no GUI toggle:

```bash
Disable crash reporter
defaults write com.apple.CrashReporter DialogType none

Disable remote Apple Events
sudo systemsetup -setremoteappleevents off

Disable remote login (SSH) if not needed
sudo systemsetup -setremotelogin off

Disable Bluetooth if not in use
sudo defaults write /Library/Preferences/com.apple.Bluetooth ControllerPowerState -int 0

Disable Wake for Network Access
sudo pmset -a womp 0

Clear Siri voice and text history via System Settings
Also delete via: ~/Library/Application Support/com.apple.assistant.support/

Disable Spotlight from indexing external drives
sudo defaults write /.Spotlight-V100/VolumeConfiguration Exclusions -array "/Volumes"

Disable sending diagnostic data from Terminal
defaults write com.apple.Terminal NSQuitAlwaysKeepsWindows -bool false
```

FileVault

System Settings → Privacy & Security → FileVault: Turn On.

FileVault encrypts the entire startup disk. On Apple Silicon and T2 Macs this is hardware-accelerated and has no perceptible performance impact.

Choose "Create a recovery key and do not use my iCloud account" for the recovery option. this keeps the recovery key local rather than escrowed with Apple.

Store the recovery key in your password manager or a printed document in a physically secure location.

Audit Installed Apps and Login Items

Login items that run at startup

System Settings → General → Login Items:
- Review everything in "Open at Login". remove anything unfamiliar
- Review "Allow in Background". these items run silently. Remove anything you don't recognize.

Third-party apps

Periodically audit installed applications:
```bash
List all apps with their TCC (privacy permission) database entries
sudo sqlite3 /Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, allowed FROM access ORDER BY service"
```

This shows every app that has requested any system permission and whether it was granted.

Related Reading

- [Privacy-Focused Web Browser Comparison 2026](/privacy-browser-comparison-2026/)
- [How to Audit Chrome Extensions for Privacy](/audit-chrome-extensions-privacy-guide/)
- [Two-Factor Authentication Setup 2026](/two-factor-authentication-setup-2026/)
- [Claude vs ChatGPT for Drafting Gdpr Compliant Privacy](https://bestremotetools.com/claude-vs-chatgpt-for-drafting-gdpr-compliant-privacy-polici/)
- [AI Tools for Automating Cloud Security Compliance Scanning](https://bestremotetools.com/ai-tools-for-automating-cloud-security-compliance-scanning-i/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Related Articles

- [How to Configure macOS Privacy Settings 2026](/how-to-configure-macos-privacy-settings-2026/)
- [macOS Network Privacy Settings Complete Guide 2026](/macos-network-privacy-settings-complete-guide/)
- [Windows 10 Privacy Settings Complete Checklist](/windows-10-privacy-settings-complete-checklist/)
- [iOS Privacy Settings Complete Walkthrough Every Toggle](/ios-privacy-settings-complete-walkthrough-every-toggle-explained/)
- [macOS Privacy Settings For Remote Workers 2026](/macos-privacy-settings-for-remote-workers-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

Frequently Asked Questions

How do I prioritize which recommendations to implement first?

Start with changes that require the least effort but deliver the most impact. Quick wins build momentum and demonstrate value to stakeholders. Save larger structural changes for after you have established a baseline and can measure improvement.

Do these recommendations work for small teams?

Yes, most practices scale down well. Small teams can often implement changes faster because there are fewer people to coordinate. Adapt the specifics to your team size, a 5-person team does not need the same formal processes as a 50-person organization.

How do I measure whether these changes are working?

Define 2-3 measurable outcomes before you start. Track them weekly for at least a month to see trends. Common metrics include response time, completion rate, team satisfaction scores, and error frequency. Avoid measuring too many things at once.

Can I customize these recommendations for my specific situation?

Absolutely. Treat these as starting templates rather than rigid rules. Every team and project has unique constraints. Test each recommendation on a small scale, observe results, and adjust the approach based on what actually works in your context.

What is the biggest mistake people make when applying these practices?

Trying to change everything at once. Pick one or two practices, implement them well, and let the team adjust before adding more. Gradual adoption sticks better than wholesale transformation, which often overwhelms people and gets abandoned.

{% endraw %}
