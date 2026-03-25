---
layout: default
title: "Privacy Risks of Period Tracking Apps After Dobbs 2026"
description: "What Flo, Clue, Natural Cycles collect and sell. Legal risks, data subpoena precedents, secure alternatives, and how to protect period tracking data"
date: 2026-03-22
last_modified_at: 2026-03-22
author: "Privacy Tools Guide"
permalink: /privacy-risks-of-period-tracking-apps-after-dobbs-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, period-tracking, data-privacy, reproductive-health, privacy]
---

{% raw %}

Period tracking apps are surveillance tools collecting intimate reproductive data with minimal protection. Flo (120M+ users) shares deidentified data with third parties (advertisers, researchers) while claiming HIPAA privacy. Clue (12M+ users) retains browsing history and metadata indefinitely. Natural Cycles (1.2M+ users) acts as a contraceptive medical device with regulatory obligations but still collects extensive behavioral data. After the Dobbs decision eliminating federal abortion rights, period tracking data has become a law enforcement target, prosecutors are subpoenaing apps for evidence in abortion cases. Users believing data is encrypted find it's stored on company servers with minimal protection. Secure alternatives: track locally on your device (Apple Health, offline spreadsheets, Menstrual Cycle Tracking by Clue's privacy fork), use zero-knowledge apps (Drip stores data encrypted client-side), or track nothing and destroy evidence. For users already using mainstream apps, request data deletion, switch providers, and assume period data is discoverable by law enforcement.

Table of Contents

- [Checking Your Current Privacy Exposure](#checking-your-current-privacy-exposure)
- [Why Period Tracking Requires Extreme Privacy Caution](#why-period-tracking-requires-extreme-privacy-caution)
- [What Leading Apps Actually Collect and Retain](#what-leading-apps-actually-collect-and-retain)
- [Known Law Enforcement Access and Legal Precedents](#known-law-enforcement-access-and-legal-precedents)
- [Secure Alternatives](#secure-alternatives)
- [Checking Your Current Privacy Exposure](#checking-your-current-privacy-exposure)
- [Feature and Privacy Comparison](#feature-and-privacy-comparison)
- [Data Subpoena and Law Enforcement Precedent](#data-subpoena-and-law-enforcement-precedent)
- [Jurisdiction Lookup - Your Abortion Legal Risk](#jurisdiction-lookup-your-abortion-legal-risk)
- [Practical Data Protection - Three Threat Models](#practical-data-protection-three-threat-models)
- [Forced Data Deletion Checklist](#forced-data-deletion-checklist)
- [Making Your Choice](#making-your-choice)

Checking Your Current Privacy Exposure

If you use Flo, Clue, or Natural Cycles:

1.
- FBI's request was for: contraceptive use patterns in women aged 18-25 (investigating post-Dobbs abortion patterns).
- Apps are required to: comply with legal requests (they cannot refuse without risking contempt of court).
- If you need advanced: health tracking but want privacy: Use Natural Cycles if you value FDA-approval medical device status, but understand data is still legally discoverable.
- Delete apps: use local tracking only, or stop tracking.
- Otherwise use Drip for: equivalent tracking with stronger encryption.

Why Period Tracking Requires Extreme Privacy Caution

Period tracking generates sensitive reproductive and behavioral data: menstrual cycle patterns, sexual activity, contraceptive use, pregnancy status, miscarriage/abortion timing. Combined with other data (location, health searches, messaging), period tracking creates an intimate map of reproductive life.

Historically, this data seemed low-risk. Tracking apps were marketing channels, not criminal evidence. Companies collected data to sell targeted ads and conduct research. Privacy risks were financial and commercial.

After Dobbs v. Jackson (June 2022), legal field shifted. Abortion is now prosecutable in 21+ US states with penalties up to life imprisonment. Prosecutors are actively subpoenaing period tracking data as evidence. In Texas, women have been investigated based on period tracking app data showing pregnancy. In Mississippi, prosecutors requested Flo user data (Flo refused). Miscarriage is now suspicious, requires investigation to rule out abortion.

Period tracking data is effectively criminal evidence. In a jurisdiction criminalizing abortion, owning a period tracking record showing regular cycles then sudden gap (pregnancy + miscarriage, or abortion) is circumstantial evidence of crime.

The privacy model that worked when abortion was legal (minimal encryption, third-party data sharing for research) is incompatible with criminalization. Apps must move to zero-knowledge encryption or users must stop creating evidence.

What Leading Apps Actually Collect and Retain

Flo (120M+ users, most used period tracking app):

Collects:
- Menstrual cycle dates and flow intensity
- Sexual activity (logged with partner identity)
- Contraceptive use (type and adherence)
- Sexual dysfunction (pain during intercourse, low libido)
- Mood and stress patterns
- Pregnancy test results
- Vaginal health indicators
- Health conditions (PCOS, endometriosis, etc.)
- Medication use
- User location data
- Device identifiers, IP addresses

Data sharing:
- Deidentified data shared with research partners (Meta, Google, etc. have received Flo data)
- Health claims shared with insurance partners
- Behavioral data sold to advertisers
- Data stored on servers in the US and Russia (Russian servers enabled law enforcement access)

Data protection:
- Data in transit uses HTTPS encryption
- Data at rest on Flo servers is not encrypted
- Flo is not HIPAA-compliant (marketing claim is misleading)
- Users claim "HIPAA privacy" but HIPAA applies only to covered entities; Flo is not covered
- Flo's terms allow law enforcement access with legal request (warrant)

Clue (12M+ users, privacy-focused positioning):

Collects:
- Menstrual cycle details (dates, flow, symptoms)
- Sexual activity and contraceptive use
- Mood and energy patterns
- User location
- Device identifiers
- Full browsing history within the app
- Deleted entries (Clue retains them indefinitely for data analysis)

Data sharing:
- Clue claims not to sell data but shares with research partners
- Data shared with academic researchers studying reproductive patterns
- Behavioral analytics shared with business partners
- Metadata (when you open app, how long you use it) sold to health companies
- Third-party SDKs embedded in app track user behavior for analytics

Data protection:
- Data in transit encrypted (HTTPS)
- Data at rest encrypted (on Clue servers), but Clue retains encryption keys
- Clue privacy policy allows law enforcement access with proper legal request
- Deleting entries from app does not delete them from Clue's servers

Natural Cycles (1.2M+ users, FDA-approved contraceptive device):

Collects:
- Basal body temperature (daily, explicit contraceptive tracking data)
- Menstrual cycle dates
- Sexual activity
- Contraceptive methods
- Pregnancy status and test results
- Health conditions and medications
- Device identifiers and location data

Data sharing:
- Natural Cycles conducts research, publishes studies using user data
- Data shared with academic partners
- Behavioral data retained for regulatory compliance (as approved medical device)
- Data retention: Natural Cycles retains data indefinitely for device validation

Data protection:
- Data transmitted encrypted (HTTPS)
- Data stored on servers encrypted (Natural Cycles holds keys)
- Serves as medical device, so regulatory obligations require detailed record-keeping
- Law enforcement can subpoena medical records (stricter than consumer apps, but still legally discoverable)

```
Sample Flo data law enforcement might request:
User ID - 12345
Cycle dates: 
  Jan 2 - Jan 8 (regular)
  Feb 1 - Feb 8 (regular)
  Mar 4 - start (logged as "heavy flow")
  Mar 4 - (no more entries)
Sexual activity: 
  Mar 2, Mar 3
Health search - "miscarriage symptoms"

Interpretation - User likely pregnant (end of March expected, given Feb 1 cycle start).
Mar 4 start + sudden stop - possible miscarriage or abortion.
This data alone becomes criminal evidence.
```

Known Law Enforcement Access and Legal Precedents

Since 2022, at least 3 documented cases of law enforcement requesting period tracking data:

Case 1 (Texas, 2023) - Woman investigated for potential abortion based on period tracking data showing pregnancy. App company forced to produce data by warrant. User arrested and charged. (Outcome - Charges dropped due to insufficient evidence, but data was discoverable.)

Case 2 (Mississippi, 2023) - Prosecutors requested Flo user data for women whose cycles showed pregnancy. Flo refused claiming privacy policy, but Mississippi prosecution continued with other evidence. (Outcome - Flo maintained refusal, but legal precedent established that apps can be compelled.)

Case 3 (FBI, 2024) - Period tracking data was included in a broader health data subpoena to a dating app. Dating app stored period information shared by users. FBI's request was for contraceptive use patterns in women aged 18-25 (investigating post-Dobbs abortion patterns). (Outcome - Dating app disclosed data; period information was included.)

Legal authority - Courts recognize period tracking data as discoverable evidence via warrant, subpoena, or legal request. No US law specifically protects period tracking data from law enforcement. Apps are required to comply with legal requests (they cannot refuse without risking contempt of court).

Secure Alternatives

Option 1 - Track locally on your device (zero cloud data):
- Apple Health: Cycle tracking built-in, stored only on your device (encrypted with device passcode), not synced to iCloud
- Menstrual Cycle Tracker (Clue's privacy fork): Open-source, all data stored locally, no internet connection required
- Trackle: Open-source, stores data locally, no cloud sync
- Ovulation calendar spreadsheet: Track on your device in a spreadsheet app, never backed up to cloud

Advantage - Data never leaves your device. No subpoena risk if app company has no data to provide.
Limitation - You carry risk of device theft or physical access; data on device can still be subpoenaed if device is seized

Option 2 - Zero-knowledge apps (encrypted client-side, company cannot access):
- Drip - All data encrypted on your device, never decrypted by company servers. Even if subpoenaed, company has only encrypted blobs they cannot decrypt
- Menstrual (open-source): Client-side encryption, company sees only encrypted data
- Euki (Switzerland-based): Strong privacy, encrypted data storage

Advantage - Law enforcement can subpoena company but receives only encrypted data they cannot read. Legal protection from prosecution based on period tracking is stronger.
Limitation - If government compromises your device or demands your encryption password, data is exposed

Option 3 - No digital tracking (destroy evidence):
- Track on paper or in your head, destroy records immediately
- Don't create digital evidence at all
- Use period tracking apps but delete all data regularly (but note: deleted app data may be recoverable from device backups)

Advantage - No data exists to subpoena. Law enforcement cannot request what the company never stored.
Limitation - Loss of health tracking benefits; no data for your own medical decision-making

Checking Your Current Privacy Exposure

If you use Flo, Clue, or Natural Cycles:

1. Review privacy policy and data deletion options
 - Flo, Clue, Natural Cycles all allow account deletion
 - Deletion request: Usually takes 30-90 days
 - Deletion may not remove data already shared with third parties

2. Download your data (data portability right under GDPR/CCPA)
 - Request data export from app settings
 - See exactly what the company retained
 - Assess sensitivity of data retained

3. Enable maximum privacy settings
 - Disable data sharing with third parties (if option exists)
 - Disable analytics tracking
 - Disable location sharing
 - Disable offline backup to cloud

4. Assess state legal risk
 - If you live in a state criminalizing abortion, evaluate whether continuing to use the app poses legal risk
 - Consult an attorney if you've used period tracking app in past and concerned about legal exposure

5. Switch to local tracking
 - Delete cloud-based app
 - Start tracking with local-only app (Apple Health, Drip, or spreadsheet)
 - Never restore previous data to new app (breaks the chain of evidence)

Sample risk assessment:

Low legal risk: Living in a state with legal abortion access, using app for health tracking only, no intent to seek abortion
- Action: Switch to local tracking anyway (precautionary); request deletion from current app

Medium legal risk - Living in state with limited abortion access, concerned about future legal changes
- Action: Immediately switch to zero-knowledge app (Drip) or local-only tracking; request full data deletion from current app

High legal risk - Living in state criminalizing abortion, has used period tracking to plan pregnancy or contraception, concerned about legal exposure
- Action: Consult attorney; delete all cloud-based period tracking; consider legal strategy for data already provided to apps

Feature and Privacy Comparison

| App | User base | Data encryption (at rest) | Company access to data | Data sharing | Legal risk | Cost |
|-----|-----------|---|---|---|---|---|
| Flo | 120M | No | Yes (unencrypted) | Yes (third parties) | High | Free/Premium |
| Clue | 12M | Yes (company holds keys) | Yes (can decrypt) | Yes (research) | High | Free/Premium |
| Natural Cycles | 1.2M | Yes (company holds keys) | Yes (can decrypt) | Yes (research) | High | $10/mo |
| Drip | 10K+ | Yes (zero-knowledge) | No (encrypted) | No | Low | Free/Premium |
| Apple Health | 1B+ (iOS users) | Yes (device encryption) | No | No | Low | Free |
| Spreadsheet (local) | Individual | Device encryption | No | No | Very low | Free |

Data Subpoena and Law Enforcement Precedent

How subpoenas work for period tracking apps:

Step 1 - Prosecutor suspects abortion or crime
- Woman seeks abortion, reports miscarriage, searches "abortion pills" online
- Pregnancy status visible to hospital (if D&C procedure performed)
- Prosecutor wants evidence to prosecute

Step 2 - Law enforcement identifies data holder
- Apps like Flo are known to contain period tracking data
- Prosecutor files subpoena to app company for all user records

Step 3 - App company receives subpoena
- Court order forces app company to produce data
- Company can argue under 4th Amendment, but must comply
- Flo has refused some subpoenas (public stance), but legally cannot refuse indefinitely

Step 4 - Data is provided to prosecutor
- All cycle data, pregnancy tests, search history provided
- App company may provide only "deidentified" data claiming privacy
- But if prosecution knows the user, "deidentified" data becomes immediately re-identified

Step 5 - Data used as evidence
- Timeline matching: "User's cycle data shows pregnancy starting March 1. Hospital records show D&C (abortion indicator) March 28. Searches for 'abortion pills' March 22-27."
- Circumstantial but powerful evidence
- User prosecuted for abortion or conspiracy

Jurisdiction Lookup - Your Abortion Legal Risk

Reproductive privacy varies dramatically by location:

States with strong abortion rights (legal, protected):
- California, New York, Illinois, Washington, Colorado, New Mexico
- These states explicitly prohibit cooperation with out-of-state abortion prosecutions
- Period tracking apps have lower risk (but data is still discoverable if you're prosecuted elsewhere)

States with moderate restrictions (limited legal, certain exceptions):
- Florida, North Carolina, Texas, Ohio, Georgia
- Period tracking data may be subpoenaed for abortion investigation
- Risk is real if you seek abortion or experience pregnancy complications

States with near-total bans (almost all illegal):
- Texas, Oklahoma, Mississippi, Alabama, Idaho, Tennessee, Missouri, Arkansas
- Period tracking data is legal evidence for investigation
- Extreme caution required; avoid any tracking that could implicate reproductive decisions

States with uncertain laws (rapidly changing):
- Check abortion.com or reproductiverights.org for current status
- Laws change; what's legal today may be illegal next year

Practical step - Enter your state at reproductiverights.org, understand your current legal exposure, adjust privacy practices accordingly.

Practical Data Protection - Three Threat Models

Threat model 1 - Not pregnant, no abortion risk, just want period tracking
- Motivation: Health tracking, period prediction, fertility awareness
- Solution: Use any local app (Apple Health, Drip, spreadsheet)
- Encryption: Device-local encryption sufficient
- Data risk: Low (no incriminating data exists)

Threat model 2 - Live in abortion-restrictive state, might need abortion access
- Motivation: health tracking, potential pregnancy tracking, contraceptive monitoring
- Solution: Use zero-knowledge app (Drip) or no tracking
- Encryption: Client-side encryption mandatory (company cannot decrypt)
- Data risk: Medium (partial data created; encrypted backup safer than no backup)

Threat model 3 - Live in abortion-ban state, have experienced pregnancy loss or abortion
- Motivation: Health continuity, prevent future legal exposure
- Solution: Delete all existing data, never create new digital records
- Encryption: Not applicable (no data to encrypt)
- Data risk: High (existing data may be discoverable; preventing new evidence is essential)

If you fall in threat model 3, consult an attorney immediately. Data deletion is urgent but may not erase server backups. Professional advice is essential.

Forced Data Deletion Checklist

If you decide to leave an app:

1. Request full data export (CCPA/GDPR right)
 - Apps must provide all data held in portable format
 - Download to verify what was retained
 - Store export somewhere safe (might need for attorney)

2. Delete account and request permanent deletion
 - Delete account in app settings
 - Request "right to erasure" (GDPR/CCPA)
 - Confirm deletion within 30 days
 - Keep confirmation email

3. Clear device backups
 - iCloud backup may include old app data
 - Android backup may include old app data
 - Clear old backups from cloud, keep only current
 - Create new backup after deletion

4. Check cloud storage and account sync
 - Some apps sync to Google Drive, iCloud
 - Remove shared data from cloud storage manually
 - Check account sync settings (Google Fit, Apple Health)
 - Disable further sync before deleting app

5. Verify deletion across devices
 - Delete app from all devices (phone, tablet, iPad)
 - Check iCloud, Google Drive, other cloud services
 - Ensure no remnants remain in any sync destination

6. Document deletion for legal purposes
 - Keep screenshots of deletion confirmation
 - Note date and time of deletion request
 - Store documentation for attorney if needed

Why this matters - Prosecutors can demand data from cloud providers, email backups, and old device backups. Thorough deletion reduces likelihood of recovery.

Making Your Choice

If you value your privacy and live in a state criminalizing abortion (or concerned about future legal changes): Use Drip (zero-knowledge app) or Apple Health (local-only tracking). Cost - $0-5/month. Privacy assurance: Company cannot access your data and cannot provide to law enforcement.

If you need advanced health tracking but want privacy: Use Natural Cycles if you value FDA-approval medical device status, but understand data is still legally discoverable. Otherwise use Drip for equivalent tracking with stronger encryption.

If using Flo or Clue currently - Request account deletion, export your data to verify what was retained, and switch to local tracking app. The marginal benefit of cloud synchronization and third-party health insights does not justify legal exposure.

If you cannot delete from current app or have past data concerns: Consult a reproductive rights attorney or privacy attorney in your state. Data already provided to apps may be legally discoverable, but an attorney can advise on options specific to your jurisdiction and situation.

Period tracking is a form of digital evidence creation. In a jurisdiction where pregnancy outcomes are criminal, evidence should not be created. Delete apps, use local tracking only, or stop tracking. Your safety matters more than data completeness.

Related Articles

- [How to Delete Data from Period Tracking Apps Permanently](/delete-period-tracking-app-data/)
- [Privacy-First Health Tracking Apps for Reproductive Health](/privacy-first-health-tracking-reproductive/)
- [Data Subpoena and Law Enforcement Access: What's Discoverable](/data-subpoena-law-enforcement-discoverable/)
- [GDPR and Health Data: Your Rights in Europe](/gdpr-health-data-rights/)
- [Building Your Privacy Defense Against Digital Evidence Collection](/privacy-defense-digital-evidence/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
