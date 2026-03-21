---
layout: default
title: "How to Set Up a Burner Phone for Protests"
description: "Practical guide to setting up a burner phone for protests. Covers phone selection, communication tools, digital evidence protection, and operational security."
date: 2026-03-21
last_modified_at: 2026-03-21
author: "Privacy Tools Guide"
permalink: /how-to-set-up-a-burner-phone-for-protests/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

A burner phone for protests provides separation between your protest participation and your primary identity, reducing law enforcement tracking risk and protecting contacts. This guide covers selecting appropriate hardware, configuring secure messaging, protecting evidence of police conduct, and operational security practices that minimize law enforcement surveillance during demonstrations.

## Why Burner Phones Matter for Protest Safety

Law enforcement attends protests with cell-site simulators (Stingrays) that intercept signals from all phones in an area. This technology identifies which phones were present at a location and when. Police can later cross-reference this data against subscriber information to target specific participants.

Additionally, if police confiscate your primary phone, burner phone separation prevents them from accessing protest contacts, messages, and communications. Digital evidence of police misconduct (photos, videos, audio) stored on a burner phone remains protected if your primary phone is seized.

Burner phones also prevent protest participation from appearing on your digital timeline. Your primary phone's location data, if searched by law enforcement, won't show protest attendance if you're not carrying it.

## Phone Selection

Buy a used phone in cash from a private seller to avoid purchase records. Avoid brand-new phones whose serial numbers are registered to you. Target older models (2-3 years old) because they're cheaper used and less likely to have biometric authentication that law enforcement could force you to unlock.

**Recommended specifications**:
- Used Android phone (Motorola, OnePlus, or Samsung), 2-3 years old
- Cost: $50-150 in cash
- 64GB+ storage for evidence collection
- Functional battery (replaced if needed)
- No biometric authentication enabled
- 4G capability (required for messaging)

**Why Android**: Android allows installation of custom ROM (operating system) for maximum control and privacy. iOS requires relying on Apple's default configurations and is harder to configure for forensic security.

**Avoid**: Brand new phones (purchase traceable to you), flagship models (distinctive to identify you with), phones with known security vulnerabilities.

**Purchasing approach**: Search local Facebook Marketplace, Craigslist, or OfferUp. Negotiate by text before meeting. Meet in a location with security cameras (retail stores, busy areas) to provide alibi if police question the purchase later. Pay cash. Do not provide your real name or contact info beyond the phone number you're using.

## SIM Card Setup

Use a prepaid SIM card purchased with cash. Bring only your burner phone and cash to the store—no ID, no wallet with identification.

**Prepaid SIM options**:
- MVNO carriers (Visible, Mint Mobile, Cricket) offer prepaid SIMs
- Convenience stores (Target, Walgreens) sell Tracfone and other prepaid cards
- Cost: $15-40 for activation, $0-30/month depending on plan

**Activation process**:
1. Buy prepaid SIM with cash
2. Do not provide ID, name, or personal information
3. Use a generic name if system requires one ("John Smith" with birth date 01/01/1950)
4. Provide a temporary email address (create throwaway email separately)
5. Do not link to your primary phone, address, or any identifying information
6. Activate minimal data plan ($5-10/month prepaid)

**Note**: US carriers are increasingly requiring ID verification for activation. Some states require ID for any SIM purchase. Research your state's requirements beforehand. If ID is required, use the burner phone setup as one-time protection with assumption that you're identified but messages remain private.

## Operating System Configuration

Before installing any apps or connecting to networks, configure the phone for privacy.

**Android Setup**:
1. Power on without connecting to Wi-Fi or cellular
2. Select "Skip" for all setup wizards and account linking
3. Go to Settings → About → Build Number (tap 7 times to enable Developer Options)
4. Go to Developer Options → Enable "USB Debugging"
5. Disable location services: Settings → Location → Off
6. Disable Wi-Fi: Settings → Wi-Fi → Off
7. Disable Bluetooth: Settings → Bluetooth → Off
8. Enable flight mode: Settings → Network & Internet → Flight Mode → On
9. Only enable cellular connection (keep flight mode, tap airplane icon, disable flight mode for cellular only)
10. Disable all Google services: Settings → Accounts → Remove any Google accounts

**App Installation**:
1. Connect to burner phone via USB cable on a secure computer
2. Install apps via USB without connecting to Google Play (prevents tracking via app install)
3. Alternative: Use F-Droid app store (open source, privacy-focused)

## Communication Apps

Select messaging apps that use end-to-end encryption and don't require phone verification.

**Signal**
- Pros: Open source, audited encryption, minimum metadata, no account required
- Cons: Works better with phone number (requires SMS verification)
- Use: Primary encrypted messaging during protests
- Installation: Download APK from official Signal website, install via USB

**Briar**
- Pros: Works without internet (uses mesh network over Bluetooth), encrypted, no phone number
- Cons: Slow, requires others also using Briar
- Use: Protest communication when cellular networks are jammed
- Installation: F-Droid app store

**Wire**
- Pros: End-to-end encrypted, minimal phone verification, European servers
- Cons: Less widely adopted than Signal
- Use: Secondary encrypted communication
- Installation: F-Droid or APK

**Jami (formerly GNU Ring)**
- Pros: Decentralized, no server required, encrypted
- Cons: Slow, poor user interface
- Use: Backup communication
- Installation: F-Droid

**Apps to avoid**: WhatsApp (Facebook owned, metadata logged), Telegram (not fully end-to-end encrypted by default), Google Messages, Facebook Messenger.

## Evidence Collection Setup

Protect photos and videos of police conduct with automatic encryption and secure backup.

**Camera Configuration**:
1. Use default camera app (minimal metadata leakage)
2. Disable geotagging: Camera app → Settings → Location tags → Off
3. Disable saving to cloud: Settings → Disable Google Drive, OneDrive, Dropbox

**Evidence Encryption**:
1. Install Nextcloud (encrypted cloud storage): Download APK from official source
2. Set up Nextcloud account on non-protest email
3. Configure automatic photo backup with client-side encryption
4. Photos backup automatically and encrypted without your phone seeing plain content

**Alternative: Local Encryption**:
1. Install VeraCrypt (open source disk encryption)
2. Create encrypted volume for evidence storage
3. Mount only when backing up evidence
4. Keep volume unmounted during protest (evidence protected even if phone seized)

## Operational Security During Protest

**Phone Management**:
- Carry only the burner phone (leave primary phone at home)
- Keep battery charged (power down if captured by police)
- Memorize emergency contact phone numbers (don't store numbers on phone)
- Avoid personal identification on your person (don't carry wallet, ID, cards)

**Communication Protocol**:
- Use encrypted messaging only (no SMS, no phone calls)
- Message groups should be small (less than 5 people) to prevent infiltration
- Don't use real names in groups (use protest role: "medic", "safety", "legal-observer")
- Clear message history daily (but NOT before backing up evidence)
- Disable message preview notifications (so screen doesn't display content to observers)

**Location Awareness**:
- Do not carry your primary phone to protest
- Leave primary phone at home during entire protest attendance
- If using cellular network, understand that burner phone location is tracked by cell towers (not as precise as GPS but still identifying you in area)
- Arrive late to protest, leave early (less likely to be identified by sustained presence in location)
- Change appearance between different protest attendance (hat, glasses, jacket)

**Law Enforcement Interaction**:
- Do not cooperate with phone searches without warrant
- If police seize phone: state "I decline to provide PIN or unlock this device"
- If arrested: state "I'm exercising my right to silence and right to legal counsel"
- Police cannot legally force you to unlock via fingerprint/face (5th Amendment protects against self-incrimination by password; courts split on biometrics)
- Even if phone is unlocked, encrypted apps (Signal, Briar) remain inaccessible without passwords

## Digital Forensics Protection

If law enforcement images your phone (forensic copy), maximize time between protest and forensic examination.

**Forensic Delay Tactics**:
1. Create large video files in encrypted folder (forensic examination takes hours per GB)
2. Fill remaining storage with dummy videos (unnecessary data increases analysis time)
3. Change phone time to future date (confuses some forensic tools)
4. Use encryption on entire phone filesystem (SlimRoms, Graphene OS on supported models)

**Better approach**: Don't store sensitive evidence on the phone long-term. Back up evidence to encrypted cloud storage immediately after protest, then delete from phone.

## Post-Protest Procedures

After protest:
1. Back up evidence photos/videos to encrypted Nextcloud
2. Verify backup succeeded
3. Delete all evidence from phone
4. Clear message history (Signal: chats → Settings → Data & Storage → Clear All)
5. Document protest observations in writing (on computer, not phone)
6. Do not connect burner phone to Wi-Fi
7. Power down burner phone and remove SIM

One week later:
1. Factory reset burner phone: Settings → System → Reset → Erase all data
2. Remove SIM
3. Destroy SIM card (cut into pieces, burn)
4. Sell phone back via Facebook Marketplace (different seller than purchase to create distance)

## Legal Context

Using a burner phone is legal. Burner phones have legitimate uses: protecting privacy, separating work/personal contexts, preventing identity theft. Simply possessing a burner phone is not illegal.

Law enforcement may investigate burner phone use as suspicious behavior if discovered. However, demonstrating that this was your only communication during protest (all encrypted) provides strongest defense that you were not orchestrating illegal conduct.

If arrested, your attorney will advise you whether to produce the phone or assert Fifth Amendment rights. Follow legal counsel, not this guide.

## Cost Summary

- Used phone: $50-150
- SIM card: $20
- Minimal plan (1 month): $10
- Total: ~$80

The cost is low relative to legal expenses if you're identified and charged. Even if you never use the burner phone setup, understanding these practices reduces anxiety about protest attendance and government surveillance.



## Related Reading

- [How to Set Up Burner Devices for Protest Organization Safety](/privacy-tools-guide/how-to-set-up-burner-devices-for-protest-organization-safety/)
- [How to Set Up Panic Button on Phone: Emergency Privacy Wipe](/privacy-tools-guide/how-to-set-up-panic-button-on-phone-emergency-privacy-wipe/)
- [How To Set Up Privacy Focused Phone Specifically For Dating](/privacy-tools-guide/how-to-set-up-privacy-focused-phone-specifically-for-dating-/)
- [How To Create Burner Email Specifically For Dating Site Regi](/privacy-tools-guide/how-to-create-burner-email-specifically-for-dating-site-regi/)
- [Android Guest Mode For Lending Phone Without Exposing Person](/privacy-tools-guide/android-guest-mode-for-lending-phone-without-exposing-person/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
