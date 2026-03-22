---
layout: default
title: "How to Detect and Remove Stalkerware From Phone 2026"
description: "Step-by-step detection methods for Android and iOS stalkerware. Removal tools, safety planning, and technical indicators of surveillance."
date: 2026-03-22
last_modified_at: 2026-03-22
author: "Privacy Tools Guide"
permalink: /how-to-detect-and-remove-stalkerware-from-phone-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, stalkerware, mobile-security, privacy, safety, cybersecurity]
---

{% raw %}

Stalkerware (also called spy apps, spyware, monitoring software) runs silently on phones, sending real-time location, text messages, call logs, and app usage to abusers. Detection requires checking for unusual battery drain, data usage spikes, suspicious apps, high temperatures, and slow performance. iOS offers better protection than Android due to stricter app controls. Removal means factory reset (nuclear option) or using dedicated removal tools (iMazing for iOS, AVG Cleaner for Android). Before removal, gather evidence, plan safe exit, and contact domestic violence resources. This guide covers technical detection, removal methods, safety planning, and resources.

## Table of Contents

- [Understanding Stalkerware: What It Is and How It Works](#understanding-stalkerware-what-it-is-and-how-it-works)
- [Detection Methods: How to Spot Stalkerware](#detection-methods-how-to-spot-stalkerware)
- [Removal Methods: Step-by-Step](#removal-methods-step-by-step)
- [Post-Removal: Securing Your Phone Going Forward](#post-removal-securing-your-phone-going-forward)
- [Safety Planning Resources](#safety-planning-resources)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)

## Understanding Stalkerware: What It Is and How It Works

Stalkerware is commercial spyware marketed as "parental monitoring" or "employee tracking" but used to spy on intimate partners. Unlike typical malware that steals money or data, stalkerware's purpose is control and harassment.

### Common Stalkerware Apps (2026)

**Android (more vulnerable)**:
- **mSpy**: $9.99-49.99/month. Hidden app, real-time location, text/call monitoring, photo access.
- **FlexiSPY**: $24.95-199/month. Advanced: call recording, ambient recording (listens to surroundings even without active call), browser history.
- **SpyBubble**: $9.99/month. Location tracking, message interception, app usage monitoring.
- **PhoneSpector**: $19.95/month. Activity log, GPS tracking, photo/video download.

**iOS (restricted)**:
- Limited options due to Apple's strict app store rules
- Most iOS stalkerware requires jailbreak
- FaceTime exploitation, iCloud password access, or device-level access more common than installed apps

### How Stalkerware Works

**Installation**:
1. Physical access to phone (5-30 minutes required)
2. App downloaded and hidden, or installed via enterprise certificate
3. App disabled from showing in home screen
4. Runs in background with minimal visibility

**Data collection**:
```
Constant monitoring:
├─ Location (GPS, cell tower triangulation)
├─ Messages (SMS, WhatsApp, Signal, iMessage)
├─ Calls (duration, phone numbers, times)
├─ Photos and videos
├─ Browser history and search queries
├─ App usage times and patterns
├─ Contacts and call logs
└─ Keystroke logging (captures everything typed)

Data transmission:
└─ Encrypted upload to attacker's control server
   (Usually triggered every 30 minutes to hourly)
```

**Control methods**:
- Abuser logs into web dashboard to view collected data
- Can delete messages from phone remotely
- Can intercept location in real-time
- Receives alerts on specific contacts or keywords

## Detection Methods: How to Spot Stalkerware

### Method 1: Physical and Behavioral Signs

**Battery drain analysis**:

Normal phone battery:
- Loses 1-2% per hour idle
- Loses 5-10% per hour during active use

Stalkerware phone:
- Loses 5-20% per hour idle
- Phone warm to touch when idle
- Battery condition may show "Health degrading" (iOS)

Test:
```
1. Disable all apps (turn on Airplane mode)
2. Observe battery drain for 1 hour
3. If drain >3%, stalkerware likely
4. Normal idle drain: 0-1% per hour
```

**Data usage spikes**:

```
Monthly data analysis:
- Turn off WiFi for 1 hour
- Check cellular data used in Settings
- Should be <1-5MB per hour idle
- If >20MB per hour, data exfiltration likely
- Run again later in month to verify pattern
```

**Call and messaging delays**:

- Calls delay 2-3 seconds before connecting (interception processing)
- Text messages fail to send once, then retry automatically
- Small delays imperceptible to user, but consistent pattern = spyware

**Phone temperature**:

- Feel back of phone when not in active use
- Phone should be cool (body temperature ~98.6°F)
- Stalkerware processing causes warmth even when not in use
- If warm when idle, background process running

### Method 2: iOS-Specific Detection

**Check iOS restrictions**:

```
Settings → General → iPhone Storage → Apps listed by size:
- Look for unfamiliar app names (especially generic names)
- Suspicious apps often hidden with names like:
  "Settings" (duplicate)
  "System Update"
  "Battery Saver"
  "Device Manager"
```

**Monitor Bluetooth connections**:

```
Settings → Bluetooth:
- Look for unknown paired devices
- Some stalkerware uses Bluetooth for covert comms
- Unknown AirPods or devices = suspicious
```

**Check iCloud activity**:

```
Settings → [Your Name] → Password & Security:
- Review "Where You're Signed In"
- Unknown devices logged into your iCloud = suspicious
- Attacker may be using iCloud to access data
```

**iOS 16+ Indicator Icons**:

```
Watch for these persistent icons in status bar:
- Blue dot = app accessing microphone or camera
- Green dot = app accessing microphone or camera (recently)
- If you see blue dot when not using apps, check Control Center
- Blue/green dots persisting without app usage = stalkerware
```

**Network activity monitoring**:

Use iOS app like **Simulator** or **Network Analyzer**:
```
Monitor outbound connections when phone is idle:
- Legitimate Apple services (iCloud, Siri, etc.)
- Unknown IP addresses or domains = suspicious
- Persistent connections when phone not in use = stalkerware
```

### Method 3: Android-Specific Detection

**Check installed apps**:

```
Settings → Apps:
1. Enable "Show system apps" toggle
2. Look for unfamiliar app names (especially generic)
3. Sort by install date
4. Check for apps installed on same date (batch installation common)
5. Suspicious indicators:
   - No publisher listed
   - No install date shown (hidden app)
   - Hidden icon (won't appear in launcher)
```

**Check app permissions**:

```
Settings → Apps → Permissions:
- Look for apps with excessive permissions:
  - Location (constant access)
  - Camera (for ambient monitoring)
  - Microphone (call recording)
  - Contacts/Messages (data collection)
- Unusual permission combinations = stalkerware
```

**Monitor battery usage**:

```
Settings → Battery or Battery Usage:
1. Check which apps consumed battery
2. Look for unfamiliar app names consuming >5%
3. Note if high-usage app is hidden from launcher
4. Check process list (Settings → Developer Options → Running Services)
5. Look for background process not matching any visible app
```

**Check device administration permissions**:

```
Settings → Security → Device Admin Apps (or Device Administrators):
- ANY app here can delete messages, change permissions, wipe device
- Stalkerware often requires device admin
- Malicious app should NOT be in list
- If unfamiliar app is admin: THIS IS STALKERWARE
```

**Monitor data usage**:

```
Settings → Network → Mobile Network Data:
1. Note total data used this month
2. Check which apps use most data (especially when idle)
3. Stalkerware uploads data periodically (30 min to hourly)
4. If one app using 100MB+ unexpectedly = data exfiltration
```

### Method 4: Browser-Based Detection

**Check browser history and search**:

If browser history seems missing or incomplete:
- Stalkerware may be deleting history to hide presence
- Or recording searches without showing in standard history

**Look for network traffic**:

On Android with developer options enabled:
```
Settings → Developer Options → Networking:
- Use Network Monitor to watch live connections
- Look for connections to unknown servers
- Stalkerware typically connects to:
  - Known spy app C&C servers (e.g., mspy domain names)
  - Generic hosting providers (Amazon AWS, Vultr, DigitalOcean)
```

## Removal Methods: Step-by-Step

### iOS Removal: Full Factory Reset (Most Reliable)

**Before resetting**:

```
CRITICAL: Gather evidence if experiencing abuse:

1. Take screenshots of:
   - Threatening messages
   - Controlling behavior patterns
   - Location data access (if visible)
   - Unusual activities documented

2. Back up to cloud (Google Photos, OneDrive, iCloud):
   - Don't use the same iCloud account if abuser has access
   - Use separate email and recovery options
   - Enable two-factor authentication

3. Contact domestic violence resources:
   - National Domestic Violence Hotline: 1-800-799-7233
   - Text "START" to 88788
   - Online chat: thehotline.org
   - They can help safety plan before you remove phone
```

**Factory reset procedure**:

```
Step 1: Create new Apple ID with recovery email you control
- Use email address abuser doesn't know
- Enable two-factor authentication
- Create complex password
- Save recovery codes in safe location (not on phone)

Step 2: Back up critical data (optional if escaping safely)
- Photos important to you
- Contacts list
- Notes, calendar

Step 3: Perform factory reset
- Settings → General → Reset
- Tap "Erase All Content and Settings"
- Enter Apple ID password when prompted
- Device will restart and return to setup screen

Step 4: Set up as new iPhone
- Don't restore from backup (old backup may include malware)
- Sign in with new Apple ID
- Reinstall apps from fresh downloads
- Don't restore from iCloud backup if abuser had access

Step 5: Restore accounts from memory
- Add email accounts (but NOT iCloud with abuser access)
- Create new passwords for all accounts
- Enable two-factor authentication everywhere
- Change passwords on all accounts from a safe device first
```

**Time required**: 30-45 minutes

**Success rate**: 99.9% (factory reset removes all software)

### iOS Removal: Partial Method (If Factory Reset Not Possible)

If you need to keep photos/data and factory reset isn't immediately possible:

```
Step 1: Disable iCloud
- Settings → [Your Name] → Sign Out
- Tap "Keep on My iPhone" for data you want to save
- Sign back in with NEW Apple ID

Step 2: Check for enterprise apps
- Settings → General → VPN & Device Management
- Remove any "Mobile Device Management" profiles
- Remove any "Configuration Profiles"
- Delete enterprise apps

Step 3: Reset location services
- Settings → Privacy → Location Services
- Disable location services entirely (most restrictive)
- Check each app: disable location permission

Step 4: Disable sharing and access
- Settings → [Your Name] → Family Sharing → Remove if shared
- Settings → [Your Name] → Password & Security → Remove unknown devices
- Remove any unknown trusted phone numbers

Step 5: Reset network settings
- Settings → General → Reset → Reset Network Settings
- This disconnects all WiFi and Bluetooth, requiring re-pairing
- Removes any covert Bluetooth connections
```

**Important**: This method is not as thorough as factory reset. If possible, do full factory reset.

### Android Removal: Safe Mode Method

Safest method that preserves some data:

```
Step 1: Boot into Safe Mode
- Press and hold Power button
- Long-press "Power off"
- Tap "Power off" and hold until "Reboot to safe mode" appears
- Tap "Reboot to safe mode"
- Phone restarts with only system apps running

Step 2: Remove suspicious apps in Safe Mode
- Settings → Apps
- Sort by recently installed or install date
- Look for unfamiliar apps
- Tap app → Uninstall
- Repeat for all suspicious apps

Step 3: Check device admin and remove if necessary
- Settings → Security → Device Admin Apps
- Uncheck any app that looks suspicious
- Tap "Deactivate" if button appears

Step 4: Run antivirus scan
- Download Malwarebytes (free) from Google Play
- Run full system scan
- Let Malwarebytes quarantine/remove threats

Step 5: Restart normally
- Restart phone (it will exit Safe Mode)
```

**Limitation**: Malware may reinstall itself unless app source is removed.

### Android Removal: Factory Reset (Most Reliable)

```
Step 1: Back up critical data (to external storage or cloud)
- Google Photos (automatic photo backup)
- Google Drive for documents
- Export contacts to .vcf file
- Back up to cloud service controlled by you

Step 2: Create new Google Account or prepare existing account
- Use Google account you create fresh for this process
- Enable two-factor authentication
- Note the account details

Step 3: Perform factory reset
- Settings → System → Reset Options → Erase All Data
- Or Settings → Privacy → Factory Reset (varies by Android version)
- Confirm you want to erase all data
- Phone will restart and return to setup screen

Step 4: Set up Android as new
- Select your country and language
- Skip phone setup (don't restore from previous backup)
- Sign in with FRESH Google Account
- Reinstall apps from Google Play (not from old backup)

Step 5: Restore data selectively
- Add Google Photos, Google Drive accounts
- Download photos from cloud backup
- Add contacts from cloud source
- Don't restore from system backup if abuser had access

Step 6: Set strong passwords and two-factor auth
- Gmail: Enable 2FA
- All important accounts: Enable 2FA
- Change passwords from safe device first
```

**Time required**: 45-60 minutes

**Success rate**: 99.9% (equivalent to iOS factory reset)

## Post-Removal: Securing Your Phone Going Forward

After removal, implement these protections:

### iOS Security

```
1. Strong authentication:
   - 6+ character passcode (minimum)
   - Better: Face ID + passcode
   - Best: Long alphanumeric passcode (12+ characters)

2. Location protection:
   - Settings → Privacy → Location Services OFF (except Maps when needed)
   - Or set to "While Using App" for each app

3. Sharing controls:
   - Settings → [Your Name] → Family Sharing: NOT shared with abuser
   - Settings → Find My → OFF if shared with abuser
   - Settings → AirDrop → "Contacts Only"

4. Two-factor authentication:
   - Apple ID: Enable 2FA
   - Email account: Enable 2FA
   - Important accounts: Enable 2FA
   - Save recovery codes in safe location

5. Communication privacy:
   - Messages → Message Filtering → Enable "Filter Unknown Senders"
   - Mute notifications from abuser-linked accounts
   - Consider blocking abuser's phone number
```

### Android Security

```
1. Strong authentication:
   - Passcode: 8+ characters, mixed case and numbers
   - Better: Fingerprint + passcode
   - Best: Pattern lock + passcode

2. Location protection:
   - Settings → Location → OFF
   - Or set "Google Location Accuracy" OFF
   - Disable "Share My Location" with Google

3. Google account security:
   - Google Account → Security → Review devices and activity
   - Remove unknown devices
   - Review app permissions
   - Enable 2-Step Verification

4. Permission controls:
   - Settings → Apps → Permissions
   - Camera: OFF for all except trusted apps
   - Microphone: OFF for all except calling app
   - Location: OFF for all apps
   - Contacts: Only for trusted apps
   - Messages: Only for trusted apps

5. Device admin review:
   - Settings → Security → Device Admin Apps
   - NOTHING should be here except legitimate Samsung/Google apps
   - Remove anything unfamiliar
```

## Safety Planning Resources

If stalkerware was found, this is domestic violence. Contact resources:

**National Domestic Violence Hotline**:
- Phone: 1-800-799-7233
- Text: "START" to 88788
- Online chat: thehotline.org
- 24/7, confidential, free

**What they provide**:
- Safety planning assistance
- Local resource referrals
- Information about legal options
- Emotional support

**Safety planning considerations**:

Before removing stalkerware:
```
1. Can you remove phone safely without abuser noticing?
   - Consider timing (when abuser not watching)
   - Abuser may contact you asking where you are if tracking enabled

2. Do you have safe place to go?
   - Domestic violence shelter
   - Friend or family willing to house you
   - Hotel with cash (not credit card)

3. Have you secured important documents?
   - ID, passport, birth certificate (copy)
   - Insurance documents
   - Financial account numbers
   - Legal custody documents (if children involved)

4. Have you planned communication?
   - New phone number (abuser doesn't know)
   - New email address (abuser doesn't know)
   - Communication plan with trusted contacts only

5. Legal considerations:
   - Contact domestic violence attorney (usually free consultation)
   - Obtain protective order if threatened
   - File police report if crimes occurred
```

## Common Mistakes to Avoid

**Mistake 1: Removing phone without safety plan**

Don't: Remove stalkerware without telling anyone or making exit plan
Do: Contact domestic violence hotline, create safety plan, then remove

**Mistake 2: Using same backup/account after reset**

Don't: Factory reset, then sign back into iCloud or Google account with abuser's password
Do: Create fresh accounts with passwords only you know

**Mistake 3: Keeping old phone after removal**

Don't: Keep stalkerware phone and add it to family plan or shared accounts
Do: Destroy old phone or give to law enforcement as evidence

**Mistake 4: Not changing all passwords**

Don't: Remove stalkerware but keep using same email/social media passwords
Do: Change passwords for all accounts from safe device

**Mistake 4: Not enabling two-factor authentication**

Don't: Set up new phone without 2FA on accounts
Do: Enable 2FA on email, phone carrier account, social media, banking

## Related Articles

- [Domestic Violence Safety Planning for Digital Privacy](/privacy-tools-guide/domestic-violence-safety-planning-digital-privacy/)
- [How to Detect Hidden Tracking Apps on Your Phone](/privacy-tools-guide/detect-hidden-tracking-apps-phone/)
- [Smartphone Security for Abuse Survivors](/privacy-tools-guide/smartphone-security-abuse-survivors/)
- [Secure Messaging Apps for Abuse Survivors](/privacy-tools-guide/secure-messaging-abuse-survivors/)
- [How to Secure Your Accounts From Partner Surveillance](/privacy-tools-guide/secure-accounts-partner-surveillance/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

{% endraw %}

Built by theluckystrike — More at [zovo.one](https://zovo.one)
