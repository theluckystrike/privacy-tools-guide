---
layout: default
title: "Privacy-Focused Alternatives to Google Analytics"
description: "Compare Plausible, Umami, Matomo, and Fathom as Google Analytics replacements. Covers self-hosted setup, GDPR compliance, and feature gaps."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: privacy-analytics-alternatives-google
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Google Analytics collects detailed behavioral data about your visitors and sends it to Google's servers, where it feeds ad targeting systems. For many sites, this means you're handing your audience data to a competitor. Privacy-focused alternatives give you the traffic insights you actually need without the data extraction.

## What You Lose and What You Keep

Before switching, be honest about what you use from Google Analytics:

**Commonly used (all alternatives provide):**
- Pageviews and unique visitors
- Traffic sources (referrers, UTM parameters)
- Top pages by traffic
- Country/region data
- Device and browser breakdown
- Custom events

**GA4 features most sites never actually use:**
- Deep funnel analysis and cohort reports
- Predictive metrics (powered by GA's data aggregation)
- Cross-device tracking (requires Google's identity graph)
- Audience segments for Google Ads integration

If you're not running Google Ads campaigns and don't need cross-device identity resolution, you won't miss what the alternatives don't have.

## The Four Main Alternatives

### 1. Plausible Analytics

**Website:** `https://plausible.io`
**Model:** Hosted (~$9/month) or self-hosted (free)
**Privacy:** No cookies, no personal data, GDPR/CCPA compliant by default

Plausible's script is 45x smaller than Google Analytics (< 1KB vs ~45KB). It doesn't track individuals — metrics are aggregated.

**Self-hosted setup with Docker:**

```bash
git clone https://github.com/plausible/community-edition
cd community-edition

# Generate secret keys
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1

# Edit docker-compose.yml with your domain and keys
cp plausible-conf.env.example plausible-conf.env
nano plausible-conf.env
```

Edit `plausible-conf.env`:

```
BASE_URL=https://analytics.yourdomain.com
SECRET_KEY_BASE=your-64-char-generated-secret
DATABASE_URL=postgres://postgres:postgres@plausible_db:5432/plausible_db
```

Start:

```bash
docker compose up -d
```

Add to your site:

```html
<script defer data-domain="yourdomain.com" src="https://analytics.yourdomain.com/js/script.js"></script>
```

**Pros:** Dead simple, beautiful UI, no cookie banner required, excellent privacy defaults
**Cons:** Less granular segmentation than GA, no session replay, paid hosted plan

### 2. Umami

**Website:** `https://umami.is`
**Model:** Hosted (free tier available) or self-hosted (free, MIT license)
**Privacy:** No cookies by default, no PII collected

Umami is open source and the most popular self-hosted option. It supports multiple sites from one installation with a clean multi-site dashboard.

**Self-hosted with Docker Compose:**

```yaml
version: '3'
services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://umami:umami@db:5432/umami
      DATABASE_TYPE: postgresql
      APP_SECRET: your-random-secret-here
    depends_on:
      - db
    restart: always

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - umami-db-data:/var/lib/postgresql/data
    restart: always

volumes:
  umami-db-data:
```

```bash
docker compose up -d
```

Default login: `admin` / `umami`. Change immediately.

Add tracking script:

```html
<script async src="https://analytics.yourdomain.com/script.js" data-website-id="YOUR-SITE-UUID"></script>
```

**Track custom events:**

```javascript
umami.track('signup-button-click', { plan: 'pro' });
```

**Pros:** Free, open source, multi-site, custom events, good UI
**Cons:** Requires database maintenance, less polished than Plausible

### 3. Matomo (Formerly Piwik)

**Website:** `https://matomo.org`
**Model:** Self-hosted (free) or cloud (~$23/month)
**Privacy:** Configurable — can match or exceed GA privacy controls

Matomo is the most feature-complete alternative. It has session recordings, heatmaps, A/B testing, and goal funnels. It can import Google Analytics data during migration. It's also the heaviest option.

**Install Matomo on a VPS:**

```bash
# Download and extract
wget https://builds.matomo.org/matomo-latest.zip
unzip matomo-latest.zip -d /var/www/
chown -R www-data:www-data /var/www/matomo

# Create database
mysql -u root -p
CREATE DATABASE matomo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'matomo'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON matomo.* TO 'matomo'@'localhost';
FLUSH PRIVILEGES;

# Configure nginx/apache to serve /var/www/matomo
# Then complete setup via web wizard at https://yourdomain.com/matomo
```

**Privacy configuration in Matomo:**

Settings > Privacy:
- Anonymize IPs: Last 2 bytes (e.g., 192.168.x.x)
- Anonymize User ID
- Data retention: 13 months (or less)
- Respect DoNotTrack: Enable

```php
// Disable tracking in PHP (for logged-in users)
if (is_user_logged_in()) {
    MatomoTracker::$doNotTrack = true;
}
```

**Pros:** Most complete feature set, full data ownership, migration path from GA4
**Cons:** Complex setup, resource intensive, requires ongoing maintenance

### 4. Fathom Analytics

**Website:** `https://usefathom.com`
**Model:** Hosted only (~$14/month)
**Privacy:** No cookies, no personal data, EU-isolated servers available

Fathom is the simplest option — hosted only, no self-hosting. It routes traffic through EU servers to avoid Schrems II issues for European visitors. The script is 2.3KB.

Add to your site:

```html
<script src="https://cdn.usefathom.com/script.js" data-site="XXXXXXXX" defer></script>
```

**Pros:** Fastest setup (5 minutes), excellent EU privacy compliance, no GDPR banner required
**Cons:** Hosted only, most expensive option, limited features

## Comparison Table

| | Plausible | Umami | Matomo | Fathom |
|---|---|---|---|---|
| Self-hosted | Yes | Yes | Yes | No |
| Cookie required | No | No | Optional | No |
| GDPR compliant | Yes | Yes | Configurable | Yes |
| Custom events | Yes | Yes | Yes | Yes |
| Funnel analysis | Basic | No | Yes | No |
| Session recording | No | No | Yes (add-on) | No |
| Free tier | Self-hosted | Self-hosted + cloud | Self-hosted | No |
| Script size | <1KB | ~2KB | ~30KB | 2.3KB |

## Migrating from Google Analytics

1. Add your new analytics snippet alongside GA for 2-4 weeks to compare numbers
2. Verify the new tool is capturing the data you care about
3. Export any historical GA data you want to keep (Google Takeout or GA4 export to BigQuery)
4. Remove GA snippet
5. If GDPR applies: update your privacy policy, remove the GA-related cookie consent

**Check your site doesn't still load GA via tag managers:**

```bash
curl -s https://yoursite.com | grep -E "googletagmanager|google-analytics|gtag"
```

If results appear, a tag manager or plugin is still injecting GA. Find and remove it.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does Go offer a free tier?**

Most major tools offer some form of free tier or trial period. Check Go's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Google Analytics Tracking Alternatives That Respect User](/privacy-tools-guide/google-analytics-tracking-alternatives-that-respect-user-pri/)
- [How To Configure Google Analytics Alternative For Gdpr](/privacy-tools-guide/how-to-configure-google-analytics-alternative-for-gdpr-compl/)
- [iPhone Analytics And Improvement Data What Apple Collects](/privacy-tools-guide/iphone-analytics-and-improvement-data-what-apple-collects-an/)
- [Privacy-Focused Analytics: Self-Hosted Options](/privacy-tools-guide/privacy-analytics-self-hosted-setup/)
- [Google Nest Hub Data Collection](/privacy-tools-guide/google-nest-hub-data-collection-what-information-google-capt/)
- [Best AI for Analyzing Google Analytics Data Exports](https://theluckystrike.github.io/ai-tools-compared/best-ai-for-analyzing-google-analytics-data-exports-with-pan/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
