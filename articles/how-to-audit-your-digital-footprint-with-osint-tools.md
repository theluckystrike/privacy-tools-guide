---
layout: default
title: "Using exiftool on photos:"
description: "Complete guide to finding your personal data online using open-source intelligence tools to identify exposure and privacy leaks"
date: 2026-03-20
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-audit-your-digital-footprint-with-osint-tools/
categories: [guides, security]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
tags: [privacy-tools-guide]---

{% raw %}

Your digital footprint is larger than you think. Your email appears in data breaches, your phone number is sold to marketers, your home address is in property records. This guide walks through using OSINT (Open Source Intelligence) tools to audit what data is exposed about you online—then how to minimize that exposure.

## Key Takeaways

- **Are there free alternatives**: available? Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **Check annualcreditreport.com (free credit**: report) 2.
- **Wayback Machine (archive.org) -**: Type your name or website - See snapshots of pages over time - Useful for finding very old digital footprints - Can request removal (though not guaranteed) 3.
- **Removing the most sensitive**: data 3.
- **Mastering advanced features takes**: 1-2 weeks of regular use.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced OSINT Techniques](#advanced-osint-techniques)
- [Tools Summary](#tools-summary)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Your Digital Footprint

Your digital footprint consists of:
- **Intentional data**: Social media profiles, public websites, registrations
- **Collected data**: Data brokers, marketing databases, leaked breach data
- **Public records**: Property ownership, court records, business filings
- **Metadata**: Locations tagged in photos, deleted posts cached by search engines
- **Third-party data**: What apps and websites know about your behavior

The goal of this audit is to find what's publicly accessible and either delete it or accept the exposure.

### Step 2: OSINT Tools for Personal Auditing

### Google Alerts

The simplest starting point—monitor when your name appears online.

Setup:
```
1. Go to google.com/alerts
2. Enter your full name in quotes: "John Smith"
3. Set frequency to "as it happens"
4. Enter your email to receive alerts
5. Repeat for variations: "john.smith", nickname, email addresses
```

Google Alerts will email you whenever your name appears on publicly indexed pages. This reveals new mentions you didn't know about.

### Have I Been Pwned (HIBP)

Check if your email or phone appears in known data breaches:

```
1. Visit haveibeenpwned.com
2. Enter email address
3. View all breaches that included your email
4. Check each breach for what data was exposed:
   - Some only got emails
   - Others got passwords, phone numbers, addresses
5. Repeat for:
   - All email addresses you've ever used
   - Secondary emails
   - Work email
   - Old email addresses
```

If your data appears in a breach, that information is now circulating in underground forums. You can't undo it, but you can respond:
- Check if that site had two-factor auth enabled
- If not, change password if you reuse it
- Consider that site compromised for future use

### Spokeo and Similar People Search Sites

These aggregator sites scrape public records and sell access to your information:

```
1. Visit spokeo.com, truthfinder.com, whitepages.com
2. Search your name
3. Review what data they have:
   - Phone number
   - Address (current and historical)
   - Age and relatives
   - Work history
   - Social media links

4. Most sites allow opt-out:
   - Look for "remove listing" or "privacy" link
   - May require proving identity
   - Expect to repeat this quarterly (sites re-add data)
```

This is tedious but worthwhile. One URL removal saves you on every future search.

### Reverse Image Search

Find where your photos appear online:

```
1. Google Images: Right-click photo → Search Image with Google
2. Bing Images: Similar feature
3. TinEye.com: Specialized reverse image search

Results show:
- Your photos used without permission
- Deepfakes using your likeness
- Profile pictures from social media
- Photos embedded in articles about you

If you find unauthorized use:
- Deepfakes: Report to platform immediately
- Photos: File DMCA takedown with image host
- Personal photos: Reverse search monthly to catch misuse early
```

### Social Media Audits

Manually check each social media platform:

```
Profile Audit Checklist:

Facebook:
- [ ] Review public profile visibility (Settings → Privacy)
- [ ] Check "Who can see your friends list?" (set to private)
- [ ] Review all public photos (many are indexed by Google)
- [ ] Check photo tagging permissions (disable auto-approve)
- [ ] Review active sessions (Settings → Security)
- [ ] Check login alerts are enabled
- [ ] Review apps with access to your data (revoke unused)

LinkedIn:
- [ ] Profile visibility set to private or limited
- [ ] Phone number hidden
- [ ] Email addresses not visible
- [ ] Endorsements disabled (if you prefer)
- [ ] Review connected apps

Instagram:
- [ ] Account set to private if not a public figure
- [ ] Review tagged photos (untag without your permission)
- [ ] Check location tagging (disable)
- [ ] Review story viewers
- [ ] Check connected apps and permissions

Twitter/X:
- [ ] Protected account if preferred (fewer followers see tweets)
- [ ] Review all direct message settings
- [ ] Check phone number visibility
- [ ] Review linked accounts
- [ ] Disable location tagging in tweets

Each platform:
- Disable "allow search engines to index profile"
- Remove phone number if not needed
- Remove address information
- Disable activity tracking for ads
```

### Domain Name Audits

If you own domains, they're in WHOIS records:

```
1. Visit whois.com
2. Search any domain you own
3. Results show:
   - Registrant name and address
   - Admin contact information
   - Nameserver details
   - Creation and expiration dates

To reduce exposure:
- Use WHOIS privacy service ($10-15/year)
- Most registrars offer this automatically
- Replaces your name with privacy service's name
- Still allows legitimate contact (goes through service)
```

### Social Security Number Monitoring

Your SSN is likely in multiple breaches:

```
1. Check annualcreditreport.com (free credit report)
2. Look for accounts you didn't open
3. Set fraud alerts:
   - Contact one credit bureau
   - They notify other two automatically
   - Fraud alert lasts 1 year
   - Gives you 30 days to lock new accounts

4. Consider credit freeze:
   - Prevents new accounts in your name
   - Still need to unfreeze to apply for credit yourself
   - Free through all three bureaus
   - Best option if you're not applying for credit
```

### Email Address Hunting

Find all email addresses associated with you:

```bash
# Using Hunter.io (free tier):
Visit hunter.io
Search your domain (if you have one)
Find emails associated with your name

# Using EmailHunter or Clearbit:
Search "yourname@company.com"
Get list of emails associated with your name

# Manual searching:
Google search: "yourname@" (with the @)
Check old resume PDFs you've posted
Review old business profiles
Check GitHub (if you've committed with email)
```

Once you've found all emails associated with you, run each through HIBP to find breach exposure.

### Step 3: Build an OSINT Dashboard

Create a spreadsheet tracking your digital footprint:

```
Google Sheet: "Digital Footprint Audit"

Data Point | Found Where | Status | Action | Date Removed---
--------|------------|--------|--------|-------------
[name]@gmail.com | HIBP - Adobe breach | Exposed | Changed password, enabled 2FA | 3/15/26
Phone (555-1234) | Spokeo | Exposed | Submitted removal request | 3/16/26
Home address | Whitepages | Exposed | Requested delisting | 3/16/26
LinkedIn profile | Public | Acceptable | Monitor only | -
Twitter @[handle] | Public | Acceptable | Assume permanent | -
Facebook photo | Album 2019 | Exposed | Untagged self | 3/17/26
```

Review quarterly:
- Re-check data broker sites (they re-add data)
- Update Google Alerts
- Review new social media activity
- Check HIBP for new breaches
```

## Advanced OSINT Techniques

### Metadata Extraction

Files like PDFs and Word docs contain metadata (author, creation date, edits):

```bash
exiftool photo.jpg
# Shows: Camera model, GPS coordinates, creation time

# For PDFs:
pdfinfo document.pdf
# Shows: Title, author, subject, creation date

Cleanup:
# Remove metadata from photos before sharing online
exiftool -all= photo.jpg
# Creates photo_original.jpg (backup) and removes all metadata

# Use Windows Properties or Mac "Get Info"
# Remove author and other identifying information
# Then delete original file
```

Never share original photos online—always strip metadata first.

### Cached Pages and Archives

Pages you've deleted may still be cached:

```
1. Google Cache
 - In Google search results: Click dropdown next to URL
 - Select "Cached" to see Google's last stored version
 - Old information can be found this way

2. Wayback Machine (archive.org)
 - Type your name or website
 - See snapshots of pages over time
 - Useful for finding very old digital footprints
 - Can request removal (though not guaranteed)

3. Archive.today
 - Similar to Wayback Machine
 - More difficult for websites to remove content
 - Request removal where possible
```

If you find cached content you want removed:
- Request removal from Google: support.google.com/websearch/answer/9109
- Request removal from Wayback Machine: archive.org/about/exclude.php

### API Searches and Data Broker Aggregators

Some tools sell access to aggregated data:

```
Tools that reveal data broker exposure:
- Optery.com: Shows which data brokers have your info
- DeleteMe.com: Bulk removal service ($129/year)
- PrivacyDuck.com: Similar bulk service
- 1Doc3.com: EU GDPR requests made easy

These services:
- Identify all data brokers with your info
- File removal requests automatically
- Monitor for re-listing (quarterly)
- Cost money but save massive time
```

### Step 4: Privacy Response Plan

After auditing, develop a response strategy:

```
Tier 1: High Risk (take action immediately)
- Email in breached password databases → Change password everywhere
- Home address public → Consider data broker removal
- Phone number public → Data broker removal, contact privacy
- SSN exposed → Credit freeze + fraud monitoring

Tier 2: Moderate Risk (plan removal)
- Social media profile details → Reduce visibility
- Old blog posts with personal info → Delete or privatize
- Photos with metadata → Remove metadata before resharing
- LinkedIn/professional profiles → Limit visibility

Tier 3: Acceptable Exposure (ongoing monitoring)
- Public social profiles → These are intentional
- Professional websites → Assume permanent
- Old Google cached pages → Monitor for new exposure
```

### Step 5: Ongoing Monitoring

Set up monthly audit reminders:

```
Monthly checklist (30 minutes):
1. Check HIBP for new breaches with your email
2. Review Google Alerts (any new mentions?)
3. Test one social media privacy setting
4. Spot-check one data broker site
5. Review credit report for suspicious activity

Quarterly (1-2 hours):
1. Full Google Alerts review
2. Spokeo/Whitepages check (request removal if needed)
3. Social media audit on one platform
4. Search Google Images for new photo usage
5. Check archive.org for old content
```

## Tools Summary

| Tool | Purpose | Cost | Time |
|------|---------|------|------|
| Google Alerts | Monitor mentions | Free | 5 min setup |
| HIBP | Check breaches | Free | 5 min per email |
| Spokeo | Find data brokers | Free/Paid | 10 min per site |
| DeleteMe | Bulk removal | $129/yr | 30 min (automatic) |
| Wayback Machine | Find old content | Free | 5 min per search |
| exiftool | Strip metadata | Free | 1 min per file |

### Step 6: Expectations

Completely removing yourself from the internet is unrealistic. Your goal is:
1. Knowing what's exposed (this audit)
2. Removing the most sensitive data
3. Limiting what new data is collected
4. Monitoring for unauthorized use

You'll never achieve total privacy, but you can significantly reduce your exposure. This audit is the first step.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [How To Audit Your Digital Footprint And Find All Accounts Li](/privacy-tools-guide/how-to-audit-your-digital-footprint-and-find-all-accounts-li/)
- [How To Minimize Digital Footprint Guide 2026](/privacy-tools-guide/how-to-minimize-digital-footprint-guide-2026/)
- [How To Detect Catfish On Dating Apps Using Osint Verificatio](/privacy-tools-guide/how-to-detect-catfish-on-dating-apps-using-osint-verificatio/)
- [Metadata Removal Tools Comparison 2026: ExifTool vs MAT2.](/privacy-tools-guide/metadata-removal-tools-comparison-2026/)
- [How To Check If Your Dating Profile Photos Are Being Used On](/privacy-tools-guide/how-to-check-if-your-dating-profile-photos-are-being-used-on/)

```

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
