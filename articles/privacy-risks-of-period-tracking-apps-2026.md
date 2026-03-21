---
layout: default
title: "Privacy Risks of Period Tracking Apps 2026"
description: "Analysis of privacy risks in menstrual tracking apps (Clue, Flo, Stardust, Drip). Data collection, legal risks post-Dobbs, self-hosted alternatives"
date: 2026-03-20
author: "Privacy Tools Guide"
permalink: /privacy-risks-period-tracking-apps/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of, privacy]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

After the U.S. Supreme Court overturned Roe v. Wade (June 2022), period tracking apps became high-risk surveillance tools. Apps designed to help women understand fertility now pose legal liability: pregnancy data in the wrong hands (law enforcement, state governments, anti-abortion states) can be weaponized to prosecute women seeking abortion.

This guide analyzes privacy risks in popular period tracking apps and recommends safer alternatives.

## Why Period Tracking Data Is Dangerous

**Criminal exposure**: In abortion-restrictive states, seeking abortion or miscarriage care can be prosecuted. Period tracking data proves pregnancy intention:

- Missed period logged + pregnancy test + abortion pill search → Evidence
- Shared with law enforcement via warrant → Criminal charges

**Data breaches**: Period tracking apps store intimate health data. Breaches expose pregnancy status, abortion history, fertility intentions.

**Data sales**: Some apps monetize anonymized health data to pharmaceutical companies. De-anonymization risks exist (correlate with other data).

**Government access**: Law enforcement can subpoena data without warrant (depending on state).

## Privacy Risk Analysis of Popular Apps

### Clue

**Parent company**: Clue B.V. (Berlin-based)

**Pricing**: Free (with ads); $9.99/month premium

**Data collection**:

```
Required:
- Period dates
- Cycle length
- Ovulation dates

Optional:
- Medication use
- Mood, energy, sex drive
- Symptoms (cramps, bleeding intensity)
- Sexual activity
- Pregnancy tests
```

**Privacy policy highlights**:

- Servers in Germany and USA (EU data protected by GDPR)
- Explicit: "We do not sell data"
- Encrypted in transit + at rest
- No third-party tracking (no Google Analytics)
- Optional health data sharing for research

**US legal risk**:

- Data stored in US servers subject to subpoena
- No US encryption standards prevent law enforcement access
- 2024 court case: Clue sued for illegal police data handover (pending)

**Risk level**: MEDIUM-HIGH

**Why**:

Clue's privacy policy is strong (better than most), but US server location = vulnerability. Police can obtain data via warrant. GDPR doesn't apply to US law enforcement.

### Flo

**Parent company**: Flo Health Inc. (Florida-based)

**Pricing**: Free (with ads); $9.99/month premium

**Data collection**:

```
Required:
- Period dates

Optional:
- Cycle length
- Ovulation predictions
- Sexual activity
- Contraception method
- Pregnancy tests
- Abortion (marked as "other" in app)
```

**Privacy policy highlights**:

- Servers: US-based (AWS)
- Data sharing: "Anonymized" health data sold to third parties (pharmaceutical, research)
- Third-party tracking: Google Analytics, Facebook Pixel
- Data retention: 5 years post-account deletion

**Red flags**:

- Explicitly mentions "anonymized" health data sharing
- De-anonymization risk: Correlate with other data (IP + location + health pattern)
- Google Analytics = Behavioral tracking
- Facebook Pixel = Ad targeting based on health data

**US legal risk**: HIGH

**2022 incident**: Privacy advocacy groups sued Flo for data misrepresentation. FTC investigation ongoing.

**Why Flo is risky**:

1. US-based servers = direct law enforcement access
2. Sells data to third parties
3. Health data + behavioral tracking = de-anonymization vulnerability
4. No encryption by default (settings available but not enabled)

### Natural Cycles

**Parent company**: Natural Cycles AB (Sweden-based)

**Pricing**: Free (limited); $10/month premium

**Data collection**:

```
Required:
- Basal body temperature (BBT) measurements
- Period dates

Optional:
- Sexual activity
- Contraception changes
```

**Privacy policy**:

- Servers: AWS (US and Europe)
- EU users: GDPR protected
- US users: Subject to US law enforcement
- No explicit third-party data sharing

**Risk level**: MEDIUM

**Why**: Swedish company with GDPR compliance, but US server for Americans = law enforcement vulnerability.

### Stardust Period Tracker

**Parent company**: Stardust Inc. (Israel-based)

**Pricing**: Free (with ads); $5.99/month premium

**Data collection**:

```
- Period dates
- Symptoms
- Mood, energy
- Sexual activity
- Pregnancy tests
```

**Privacy policy**:

- Servers: Israel + US
- No data sharing (appears legitimate)
- Weak encryption (standard TLS only)
- No transparency report

**Risk level**: HIGH

**Why**:

1. Israel-based = Potential Israeli government access (political/surveillance concerns)
2. Weak encryption standard
3. No transparency report (unclear what happens on government requests)
4. Smaller company = less resources for security

### Drip (Community-Focused)

**Parent company**: Open-source community

**Pricing**: Self-hosted (free); mobile app (free)

**Data collection**:

```
- Period dates
- Cycle length
- Symptoms
- Notes
```

**Privacy**:

- Open-source code (auditable)
- Fully self-hosted (data stays on your device or server)
- No third-party tracking
- No data sharing

**Risk level**: LOW

**Why**: You control your data. No company can sell it or hand it to police.

**Limitation**: Requires technical setup (self-hosting isn't simple for non-technical users).

## Comparative Risk Table

| App | Data Location | Encryption | Third-party Tracking | Data Sales | Law Enforcement Risk | Price |
|-----|---------------|-----------|----------------------|------------|---------------------|----|
| Clue | Germany + USA | Strong | None | No | MEDIUM-HIGH | Free/$10 |
| Flo | USA | Moderate | Yes (Facebook) | Yes | HIGH | Free/$10 |
| Natural Cycles | Sweden + USA | Strong | No | No | MEDIUM | Free/$10 |
| Stardust | Israel + USA | Weak | Limited | No | HIGH | Free/$6 |
| Drip | Self-hosted | Strong | None | No | LOW | Free |
| Apple Health | US (Apple) | Strong | No (encrypted) | No | MEDIUM | Free |

## Legal Risks by State (Post-Dobbs)

**States with severe abortion restrictions** (triggered laws + prosecutions):

- Alabama, Arkansas, Idaho, Kentucky, Mississippi, Missouri, North Dakota, Oklahoma, South Dakota, Tennessee, Texas, Wisconsin, Wyoming

In these states, period tracking data can be used to:

1. Prove pregnancy intention → Prosecute abortion pill use
2. Prove miscarriage attempt → Prosecute negligent homicide (proposed laws)
3. Support warrant for search/seizure of communications

**Example prosecution chain**:

```
User in Texas uses Flo app:
- Logs missed period
- Logs pregnancy test (+)
- Searches "abortion pill" (tracked by ISP)
- Orders mifepristone online (DEA tracking)

Flo data subpoenaed by Texas AG:
- Pregnancy logged
- Timing + other data = Proof of intent
- Criminal charges: Abortion violation in TX

Risk: Up to 99 years prison (Texas law)
```

**Safer states** (blue states, abortion legal):

- California, Colorado, Illinois, Minnesota, New York, Oregon, Washington

In these states, law enforcement less likely to prosecute, but **federal government could override** (if national abortion ban passed).

## Safe Alternatives to Mainstream Apps

### 1. Drip (Self-Hosted, Recommended)

**What it is**: Open-source period tracker you host yourself.

**Setup**:

Option A: **Managed hosting** (easiest)

- Go to [dripperiod.app](https://dripperiod.app/)
- Click "Self-Hosted" → Choose Vercel (free) or AWS (paid)
- Data stays in your account, not shared

Option B: **Self-host locally**

```
git clone https://github.com/drip-tracker/drip.git
cd drip
npm install
npm start
```

Runs on your laptop. Data never leaves your device.

**Privacy**:

- Open-source (audit the code yourself)
- No tracking, no ads, no data sales
- No third-party access
- Fully encrypted (if using local storage)

**Features**:

- Period tracking
- Symptom logging
- Cycle predictions
- Export data (CSV)

**Limitations**:

- No mobile app (web app is responsive)
- Requires technical setup
- No synchronization between devices

**Cost**: Free

**Risk level**: LOW

### 2. Apple Health (Native, Moderate Privacy)

**What it is**: Apple's health app (built into iPhone/iPad).

**Privacy**:

- End-to-end encrypted on device
- Data synced encrypted to iCloud
- Apple claims they can't read health data (encrypted key management)
- US-based, subject to subpoena
- No third-party tracking

**Features**:

- Menstrual tracking (cycle prediction, pregnancy mode)
- Cycle insights
- Share data with health providers

**Setup**:

1. Open Health app (iOS)
2. Cycles tab → "Get Started"
3. Log periods manually
4. App predicts ovulation, fertile days

**Privacy considerations**:

- Encrypted storage but Apple has encryption keys
- Apple could be compelled to decrypt
- Still subject to US law enforcement

**Advantages over mainstream apps**:

- Integrated (no separate account)
- No ads, no data sales
- Basic encryption
- No behavioral tracking (no Facebook Pixel)

**Risk level**: MEDIUM (better than Flo, similar to Clue)

**Cost**: Free

### 3. Euki (European-Based, Good Privacy)

**What it is**: Period tracking app from Berlin-based company.

**Privacy**:

- GDPR compliant
- Server in Germany (EU data protection)
- Encrypted in transit + at rest
- No third-party tracking
- Explicit: "No data sales"

**Features**:

- Period tracking
- Ovulation predictions
- Symptom logging
- Educational content

**Limitations**:

- No US law enforcement protection (but lower likelihood)
- Requires German infrastructure (slower US access)
- Smaller app (fewer features than Flo)

**Risk level**: MEDIUM (better than Flo, similar to Clue)

**Cost**: Free/$5/month premium

**Note**: Euki less established than Clue. Review privacy policy carefully before relying on it.

### 4. Contraceptive Apps (Temperature-Based)

**What it is**: Fertility awareness apps using basal body temperature (no symptom guessing).

**Examples**: Natural Cycles, Tempdrop (hardware + app)

**Privacy**:

- Smaller data set (temperature only, not symptoms)
- Less intimate data = Lower exploitation risk
- BBT data less directly linked to abortion intention

**Limitations**:

- Requires daily temperature measurement
- Less user-friendly than period-only trackers

### 5. Spreadsheet (Maximum Control)

**The simplest option**:

```
Date | Period | Symptoms | Notes | Mood

2026-03-01 | Start | Cramps | - | Tired
2026-03-02 | Moderate | - | - | Normal
2026-03-15 | - | - | - | -
2026-03-28 | End | - | - | -
```

Store in:
- Encrypted spreadsheet (password-protected)
- Encrypted file (VeraCrypt, 7-Zip)
- Handwritten notebook (most private)

**Advantages**:

- No company can access your data
- No digital trail (if handwritten)
- Complete control
- No tracking

**Disadvantages**:

- No automatic ovulation predictions
- Manual calculations
- No mobile app
- Limited analysis

## Recommendations by Situation

### Situation 1: Live in Abortion-Restrictive State

**Use**: Drip (self-hosted) or Apple Health

**Why**: No company server = No law enforcement access

**Setup**:

- Drip: 30 minutes to self-host
- Apple Health: Native (already on iPhone)

### Situation 2: Travel Between States

**Use**: Apple Health + encrypted backup

**Why**: Encrypted on device, harder to access while traveling

**Setup**:

1. Use Apple Health for daily tracking
2. Regularly export data (CSV)
3. Store CSV in encrypted file (7-Zip, password: 20+ chars)
4. Store encrypted file in cloud (Google Drive, OneDrive—encrypted file passwords not readable by cloud provider)

### Situation 3: Privacy-Conscious But Not Legal-Risk

**Use**: Clue (German-based, transparent, no data sales)

**Why**: Better privacy than most, GDPR compliance (though not protection from US law enforcement)

**Setup**:

1. Create account (use throwaway email)
2. Use password manager (generate unique 30-char password)
3. Enable 2-factor authentication
4. Log period only (don't log sexual activity, mood, medication—unnecessary data)

### Situation 4: Live in Safe State + Want Features

**Use**: Clue (premium) or Natural Cycles

**Why**: Strong privacy policies, not data-sales companies

**Cost**: $10/month for Clue, $10/month for Natural Cycles

## Red Flags in Privacy Policies

**Avoid apps that**:

- [ ] Explicitly mention selling "anonymized" data
- [ ] Use third-party analytics (Google Analytics, Mixpanel, Facebook Pixel)
- [ ] Retain data indefinitely post-deletion
- [ ] Don't encrypt data in transit
- [ ] US servers without encryption
- [ ] No transparency report
- [ ] Vague about government requests ("We comply with all applicable laws")

## Best Practices

**If using mainstream app** (Clue, Flo, etc.):

1. **Minimize data**: Log periods only, not sexual activity or symptoms
2. **Delete regularly**: Set auto-delete (if available) or manually delete old entries
3. **Use pseudonym**: Email account not tied to real name
4. **Strong password**: 20+ characters, randomly generated (use password manager)
5. **Enable 2FA**: Protect account from unauthorized access
6. **No sync**: Don't sync to multiple devices
7. **Delete account**: If moving to abortion-restrictive state, delete account and data

**If using self-hosted** (Drip):

1. **Backup encrypted**: Encrypt database file
2. **Offline storage**: Keep backup on encrypted external drive
3. **No cloud**: Don't store in Google Drive/iCloud unencrypted
4. **Regular review**: Periodically audit data for unnecessary information

## Legal Reality

**Important caveat**: No app is 100% safe from law enforcement if:

- Police get physical device
- NSA/FBI exploits encryption
- Federal law changes

**Best protection**: Don't log pregnancy-related data in any app if in high-risk state.

Instead:
- Use mental math (count days manually)
- Write in encrypted notes (not date-searchable)
- Handwritten notebook (no digital trail)

{% endraw %}
## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

