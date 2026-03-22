---
layout: default
title: "Privacy Risks of Fitness Trackers and Wearables"
description: "What Fitbit, Garmin, Apple Watch, and Oura Ring collect, who gets access to your health data, and practical steps to reduce wearable privacy exposure"
date: 2026-03-22
author: theluckystrike
permalink: /privacy-risks-wearables-fitness-data/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

# Privacy Risks of Fitness Trackers and Wearables

A fitness tracker knows your resting heart rate, sleep patterns, menstrual cycle, stress levels, activity at 3am, and location throughout the day. This data is more revealing than most people realize — health insurers, employers, and data brokers have significant financial interest in it. This guide covers what each device collects, who accesses it, and what you can actually do.
---

## What Fitness Trackers Collect

### Continuous Biometric Data

Modern wearables generate a continuous stream:

- **Heart rate**: Resting, active, elevated, bradycardia/tachycardia events
- **Heart rate variability (HRV)**: Correlates with stress and recovery state
- **SpO2**: Blood oxygen saturation
- **Sleep patterns**: Duration, stages (light/deep/REM), interruptions, estimated breathing
- **Skin temperature**: Baseline and deviations
- **Menstrual cycle tracking**: Cycle dates, flow, symptoms, ovulation predictions
- **GPS location**: Route history, visited locations, patterns
- **Activity data**: Steps, distance, exercise type, calories

This isn't generic health data — it's a precise physiological record that can reveal pregnancy, illness, mental health state, and addiction patterns.

---

## Company Data Practices

### Google/Fitbit

Google acquired Fitbit in 2021. Google has committed to not using Fitbit health data for ad targeting — this commitment runs until 2026 by regulatory requirement. What happens after is not specified.

**What Google/Fitbit stores:**
- Full sensor history
- Sleep logs
- GPS routes
- Connected apps data
- Google account tied to all of it

**Access**: US users are subject to CLOUD Act requests. Google received 89,000 government data requests in 2022 (combined across products).

### Garmin

Garmin had a major ransomware attack in 2020 that exposed the fragility of centralized fitness data. Garmin stores full activity and health data on their servers.

**Garmin Connect data export**: You can download a full archive, which reveals the extent of stored data:

```bash
# Request data export: Garmin Connect → Account → Data Export
# Export includes: activities, health stats, body composition, sleep logs
# Format: FIT files (GPS + biometrics), CSV (daily stats), JSON (health data)

# Parse FIT files to inspect GPS routes
pip install fitparse
python3 - << 'EOF'
import fitparse

fitfile = fitparse.FitFile("activity.fit")
for record in fitfile.get_messages('record'):
    data = {d.name: d.value for d in record}
    lat = data.get('position_lat')
    lon = data.get('position_long')
    if lat and lon:
        # FIT format uses semicircles — convert to degrees
        lat_deg = lat * (180 / 2**31)
        lon_deg = lon * (180 / 2**31)
        print(f"  {lat_deg:.6f}, {lon_deg:.6f}")
EOF
```

### Apple Watch

Apple's privacy posture is meaningfully better than Fitbit/Garmin. Health data is stored in the Health app on-device, encrypted with your iPhone's lock code.

**iCloud backup**: If iCloud Health backup is enabled WITHOUT Advanced Data Protection, Apple holds the keys and could technically access health data. With ADP enabled, end-to-end encryption means Apple cannot read it.

```
Settings → [Your Name] → iCloud → Health:
  - Enabled: data syncs to iCloud (Apple holds keys by default)
  - Disabled: data stays on device only

Settings → [Your Name] → iCloud → Advanced Data Protection:
  - Enabled: iCloud Health data is E2EE (Apple can't access)
```

Apple does share data with third-party apps you authorize (e.g., MyFitnessPal, Strava) — review what you've granted.

### Oura Ring

Oura charges a subscription ($5.99/month) and stores ring data on their servers. The data includes sleep stages, HRV, readiness score, and SpO2.

Oura's privacy policy allows data sharing with "research partners" — though with de-identified data. Oura also worked with the NBA and employers during COVID to use ring data for health monitoring — raising concerns about employer access to health data.

---

## Third-Party App Data Sharing

The wearable itself is just the start. Every app you connect to your fitness platform gets access to your data:

```bash
# Check what apps have access to Fitbit data
# Fitbit app → Account → Apps → Connected Apps
# Revoke anything you don't actively use

# Apple Health connected apps
# Health app → Sharing tab → see all apps with access
# Tap each → turn off categories you don't want to share

# Google Fit connected apps
# myaccount.google.com → Third-party apps with account access
```

Apps commonly connected to fitness data:
- Insurance apps (Vitality, John Hancock) — may affect premiums
- Employer wellness programs — explicitly designed for employer access
- MyFitnessPal, Strava, Calm — sold user data historically
- Fertility apps — extremely sensitive; many had data breach incidents

---

## Insurance and Employer Risks

**Health insurance**: Some health insurers (primarily in the US) offer premium discounts for fitness tracker use. The trade-off: they get access to your activity data. Discounts of $100-300/year in exchange for continuous health monitoring.

**Life insurance**: John Hancock's Vitality program uses Apple Watch data to adjust life insurance premiums dynamically. More steps = lower premium, potentially. But the insurer also knows when you stop exercising or your HRV drops.

**Employer wellness programs**: Programs like Virgin Pulse, Limeade, and Wellable pay employees to wear trackers. The employer or their vendor receives aggregated (and sometimes individual) health data.

```
Practical guidance:
- Never connect your personal wearable to an employer wellness program
  unless you've read their full privacy policy
- If required, use a separate cheap device for the wellness program
- Never connect your main health data to insurance without understanding
  exactly what data is shared and for how long
```

---

## Practical Privacy Steps

### Minimize Data Collection

```
Fitbit/Google:
- Disable "Move IQ" (automatic activity detection) if not needed
- Disable GPS tracking for walks if route privacy matters
- Turn off menstrual health tracking if not using it
- Auto-delete data: Fitbit doesn't have this; manually delete old activity

Apple Watch:
- Health → Browse → turn off categories you don't want tracked
- Location: Settings → Privacy → Location Services → each fitness app → When Using

Garmin:
- Garmin Connect → Account → Privacy Settings → Activity sharing: Off
- Disable: Insights, Garmin Health Reports
- Limit connected apps aggressively
```

### Use a Dumb Device for Specific Tracking

A Fitbit Inspire or a basic pedometer tracks steps without GPS, heart rate, or cloud connectivity. Gives you basic step counts without the biometric profile.

### Export and Delete

```bash
# Regularly export and delete your data
# Fitbit: Account → Data Export → request archive
# After downloading: Settings → Clear User Data

# Garmin: Account → Data Management → Delete Account (nuclear option)
# Or: selectively delete individual activities

# Apple Health export (for backup before changing phones)
# Health → Profile → Export All Health Data
```

### Local-Only Fitness Tracking

For maximum control:

```bash
# Running/cycling GPS tracking without cloud: use OSMand or OsmAnd~ (F-Droid)
# Data stays on device, exported as GPX
# No account required

# Heart rate monitoring without cloud: Polar devices have local storage
# Connect to computer via USB → download FIT files → no cloud needed

# Sleep tracking without cloud:
# SleepAsAndroid with local storage (no account mode)
# Settings → Cloud & Premium → disable all cloud features
```

---

## Data Your Device Reveals Even Without Apps

A fitness tracker with GPS that you carry everywhere creates a location history:

```bash
# Where you live (home location, most common overnight GPS point)
# Where you work (regular 9-5 GPS location on weekdays)
# Medical appointments (cluster of visits to clinic address)
# Religion/political activity (location on weekend mornings = church; protests)
# Relationship status (second location you regularly sleep at)
```

This is the pattern-of-life analysis used by intelligence agencies — and available to any party that gets access to your fitness data. A subpoena for Fitbit records has been used in US criminal cases to establish alibis and disprove them.

---

## Related Reading

- [Privacy Risks of Fitness Trackers Health Data 2026](/privacy-tools-guide/privacy-risks-of-fitness-trackers-health-data-2026/)
- [Privacy-Focused Fitness Trackers Comparison 2026](/privacy-tools-guide/privacy-focused-fitness-trackers-comparison-2026/)
- [Android Background Location Access Which Apps Track You](/privacy-tools-guide/android-background-location-access-which-apps-track-you-when/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

