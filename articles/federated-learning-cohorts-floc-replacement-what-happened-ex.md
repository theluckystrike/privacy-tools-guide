---
layout: default
title: "Federated Learning Cohorts: FLoC Replacement Explained"
description: "Federated Learning Cohorts: FLoC Replacement Explained — privacy guide covering tools, techniques, and best practices to protect your data and digital"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /federated-learning-cohorts-floc-replacement-what-happened-ex/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Google's FLoC failed due to user privacy concerns and regulatory opposition, leading to its replacement with the Topics API in 2023. Topics provides coarser interest categories updated weekly rather than persistent cohort assignments, reducing re-identification risks while still enabling interest-based advertising. Developers migrating from cookies should understand that Topics, Protected Audiences, and Attribution Reporting form the modern Privacy Sandbox—each addressing different advertising use cases with varying privacy guarantees.

## What Was Federated Learning of Cohorts?

FLoC aimed to solve a fundamental problem in digital advertising: how to target users without tracking them individually across websites. The original approach used federated learning, a machine learning technique where models train across distributed data sources without centralizing the raw data.

The core mechanism worked like this:

1. **Cohort Assignment**: Your browser would analyze your browsing history locally and assign you to a "cohort" of approximately 1,000 to 5,000 users with similar interests
2. **Cohort ID Sharing**: Websites would receive only your cohort ID, not your specific browsing history
3. **Ad Selection**: Advertisers could target specific cohorts without knowing individual user identities

Google described this as a privacy improvement because individual browsing data never left the user's device. Only the aggregated cohort membership was shared with websites and advertisers.

```javascript
// Conceptual FLoC API usage (deprecated)
async function getCohort() {
  const cohort = await document.interestCohort();
  return cohort.id; // e.g., "cohort12345"
}

// Website would then use this for ad targeting
const userCohort = await getCohort();
console.log(`User belongs to cohort: ${userCohort}`);
```

The initial implementation in Chrome featured an API that looked like this:

```javascript
// FLoC API prototype (no longer functional)
const flocId = await interestCohort.get({
  advertised: 'https://example.com',
  publisher: 'https://publisher-site.com'
});
```

## Why FLoC Failed

Despite Google's privacy-preserving intentions, FLoC faced significant criticism from multiple directions:

### Privacy Concerns

Security researchers identified several vulnerabilities:

- **Fingerprinting Risk**: Cohort IDs could serve as a new form of fingerprinting. Researchers demonstrated that combining FLoC with other browser signals could still identify individuals
- **Sensitive Categories**: Cohorts could inadvertently reveal sensitive information about users, such as health conditions or political affiliations
- **Historical Data**: The algorithm still relied on analyzing browsing history, even if that analysis happened locally

### Regulatory Pushback

Privacy regulators expressed concerns:

- The Electronic Frontier Foundation (EFF) called FLoC "the opposite of privacy"
- The UK Information Commissioner's Office raised objections during the Privacy Sandbox testing phase
- Multiple web developers implemented headers to block FLoC:

```apache
# Block FLoC in Apache
Header always set Permissions-Policy: interest-cohort=()

# Block FLoC in Nginx
add_header Permissions-Policy "interest-cohort=()";
```

### Community Rejection

Major platforms and browsers rejected FLoC:

- Brave Browser blocked FLoC by default
- Vivaldi implemented anti-FLoC measures
- Firefox and Safari never implemented FLoC
- WordPress.com automatically blocked FLoC for all sites

## The Replacement: Topics API and Privacy Sandbox

Google replaced FLoC with a redesigned approach called the Topics API, which launched in Chrome starting in 2024. The new system incorporates lessons learned from the FLoC controversy.

### How Topics API Works

The Topics API takes a simpler approach to interest-based advertising:

1. **Weekly Topic Calculation**: Chrome analyzes your browsing once per week to determine approximately 5 topics
2. **Topic Selection**: Topics are chosen from a curated taxonomy of approximately 1,400 categories
3. **One Topic Shared**: When you visit a website, the API shares one randomly selected topic from your current list
4. **No Persistent History**: Topics are recalculated weekly and older data is discarded

```javascript
// Topics API implementation (current)
async function getTopics() {
  const topics = await document.interestCohort.get();
  return topics.map(topic => topic.topic);
}

// Example output:
// [
//   { topic: "Sports/Football", taxonomyVersion: "v1" },
//   { topic: "Technology/Programming", taxonomyVersion: "v1" }
// ]

// Website usage
const userTopics = await getTopics();
console.log('Relevant topics:', userTopics);
```

### Key Differences from FLoC

| Aspect | FLoC | Topics API |
|--------|------|------------|
| Identifiers | Cohort ID (weeks of history) | Single topic (current week) |
| Persistence | Long-term assignment | Weekly refresh |
| Granularity | Thousands of cohorts | ~1,400 topics |
| Sensitive categories | Not explicitly handled | Explicitly excluded |
| Third-party access | Could be shared with third parties | Only first-party access |

### Protected Audiences API

Google also developed the Protected Audiences API (formerly FLEDGE) for remarketing use cases. This API allows advertisers to target users based on custom audience lists without revealing those lists to other parties:

```javascript
// Protected Audiences API (for remarketing)
async function joinAudience() {
  const auctionConfig = {
    seller: 'https://publisher.com',
    decisionLogicURL: 'https://seller.com/decision-logic.js',
    interestGroupBuyers: ['https://advertiser.com'],
    auctionSignals: {},
    perBuyerSignals: {}
  };

  await navigator.joinAdInterestGroup({
    name: 'custom-audience-1',
    owner: 'https://advertiser.com',
    biddingLogicURL: 'https://advertiser.com/bidding-logic.js',
    ads: [{
      renderURL: 'https://advertiser.com/ad.html',
      metadata: { adId: '123' }
    }]
  }, 30 * 24 * 60 * 60); // 30 days expiration
}
```

## Developer Implementation

For developers working with the Privacy Sandbox APIs in 2026, here's the current field:

### Detecting API Availability

```javascript
// Check if Topics API is available
const topicsAvailable = 'interestCohort' in document;

// Check Protected Audiences
const protectedAudiencesAvailable = 'joinAdInterestGroup' in navigator;
```

### Opting Users Out

```javascript
// Opt out of Topics API
document.interestCohort.doNotShare();

// Clear all interest groups (Protected Audiences)
await navigator.leaveAdInterestGroup({ name: '*', owner: '*' });
```

### Blocking These APIs Server-Side

```javascript
// Express.js middleware to disable Privacy Sandbox APIs
app.use((req, res, next) => {
  res.setHeader('Permissions-Policy', 'interest-cohort=(), geolocation=(), microphone=()');
  next();
});
```

```nginx
# Nginx configuration
add_header Permissions-Policy "interest-cohort=(), browsing-topics=()";
```

## The Current State

As of 2026, the Privacy Sandbox APIs have reached broader adoption but remain controversial. Chrome holds approximately 65% market share, and the Topics API is enabled by default. However:

- Firefox and Safari continue to not implement these APIs
- The ePrivacy Directive in Europe still requires consent for any form of tracking
- Privacy advocates recommend using browsers that block these APIs entirely

For developers, the practical guidance remains:

1. **Respect user preferences**: Implement proper consent mechanisms
2. **Use first-party data**: Build relationships where users explicitly provide information
3. **Monitor regulatory changes**: Privacy laws continue to evolve
4. **Consider alternatives**: Contextual advertising remains a viable option

The story of FLoC demonstrates how difficult it is to balance advertising business models with user privacy. While the Topics API represents an improvement over the original FLoC design, it continues to face scrutiny. Developers should stay informed about developments in this space and prioritize building trust with users through transparency and respect for their privacy choices.
---


## Measuring Topics API Impact on Your Traffic

## Table of Contents

- [Measuring Topics API Impact on Your Traffic](#measuring-topics-api-impact-on-your-traffic)
- [Alternative Advertising Approaches](#alternative-advertising-approaches)
- [Measuring Privacy Impact](#measuring-privacy-impact)
- [Compliance Framework](#compliance-framework)

### Analytics Integration

Understand how Topics API affects your website traffic and targeting:

```javascript
// Detect Topics API availability and topic information
async function detectTopicsAPI() {
    if (!document.interestCohort) {
        console.log("Topics API not available");
        return null;
    }

    try {
        const topics = await document.interestCohort();
        console.log("Current topics:", topics);

        // Log to analytics
        gtag('event', 'topics_available', {
            'topics_count': topics.length,
            'topics': topics.map(t => t.topic).join(',')
        });

        return topics;
    } catch (error) {
        console.log("Topics API error:", error);
        return null;
    }
}

// Call on page load
detectTopicsAPI();
```

### Privacy Sandbox Testing Dashboard

Build a testing environment to measure Privacy Sandbox API performance:

```html
<!-- Privacy Sandbox Testing Dashboard -->
<div id="sandbox-dashboard">
    <h2>Privacy Sandbox API Status</h2>

    <div id="topics-status">
        <h3>Topics API</h3>
        <p>Status: <span id="topics-available">Checking...</span></p>
        <p>Current Topics: <span id="current-topics">Loading...</span></p>
    </div>

    <div id="protected-audiences-status">
        <h3>Protected Audiences</h3>
        <p>Status: <span id="pa-available">Checking...</span></p>
        <p>Interest Groups: <span id="interest-groups">Loading...</span></p>
    </div>

    <div id="attribution-status">
        <h3>Attribution Reporting</h3>
        <p>Status: <span id="attr-available">Checking...</span></p>
    </div>
</div>

<script>
// Check API availability
document.addEventListener('DOMContentLoaded', async () => {
    // Topics API
    const topicsAvailable = 'interestCohort' in document;
    document.getElementById('topics-available').textContent = topicsAvailable ? 'Available' : 'Not Available';

    if (topicsAvailable) {
        try {
            const topics = await document.interestCohort();
            document.getElementById('current-topics').textContent =
                topics.map(t => t.topic).join(', ');
        } catch (e) {
            document.getElementById('current-topics').textContent = 'Error: ' + e.message;
        }
    }

    // Protected Audiences
    const paAvailable = 'joinAdInterestGroup' in navigator;
    document.getElementById('pa-available').textContent = paAvailable ? 'Available' : 'Not Available';

    // Attribution Reporting
    const attrAvailable = 'attributionReporting' in document;
    document.getElementById('attr-available').textContent = attrAvailable ? 'Available' : 'Not Available';
});
</script>
```

## Alternative Advertising Approaches

### Contextual Advertising (Privacy-Preserving)

For developers moving away from Topics API, contextual advertising provides an alternative:

```javascript
// Contextual ad targeting based on page content, not user history

function analyzePageContext() {
    // Extract keywords from page content
    const pageText = document.body.innerText;
    const keywords = extractKeywords(pageText);

    // Analyze visible elements
    const headings = Array.from(document.querySelectorAll('h1, h2'))
        .map(el => el.textContent);

    // Don't send any user data to ad server
    // Only send current page context
    loadContextualAds({
        pageKeywords: keywords,
        pageTopics: headings,
        timestamp: new Date().getTime()
    });
}

function extractKeywords(text) {
    // Simple keyword extraction (production would use better algorithm)
    const commonWords = new Set(['the', 'a', 'an', 'and', 'or', 'but']);
    return text
        .toLowerCase()
        .match(/\b\w+\b/g)
        .filter(word => word.length > 3 && !commonWords.has(word))
        .slice(0, 10);
}
```

### First-Party Data Strategy

Build audience segments using explicit user consent:

```javascript
// First-party data collection with explicit consent

class ConsentedAudienceManager {
    constructor() {
        this.audiences = new Map();
        this.consentRequired = true;
    }

    // Ask user for explicit consent
    async requestConsent() {
        return new Promise((resolve) => {
            const modal = document.createElement('div');
            modal.innerHTML = `
                <div class="consent-modal">
                    <h3>Help us personalize your experience</h3>
                    <p>We'd like to remember your preferences for:</p>
                    <ul>
                        <li>Product categories you're interested in</li>
                        <li>Preferred content types</li>
                    </ul>
                    <button id="consent-yes">Accept</button>
                    <button id="consent-no">Decline</button>
                </div>
            `;

            document.body.appendChild(modal);

            document.getElementById('consent-yes').onclick = () => {
                localStorage.setItem('audience_consent', 'true');
                resolve(true);
            };

            document.getElementById('consent-no').onclick = () => {
                localStorage.setItem('audience_consent', 'false');
                resolve(false);
            };
        });
    }

    // Build first-party audience based on user actions
    recordUserAction(action, category) {
        if (!localStorage.getItem('audience_consent')) return;

        const key = `audience_${category}`;
        const current = this.audiences.get(key) || 0;
        this.audiences.set(key, current + 1);

        // Save to localStorage
        localStorage.setItem(key, current + 1);
    }

    // Get user's audience segments
    getAudiences() {
        if (!localStorage.getItem('audience_consent')) {
            return [];
        }

        const audiences = [];
        this.audiences.forEach((count, key) => {
            if (count > 2) {
                audiences.push(key.replace('audience_', ''));
            }
        });

        return audiences;
    }
}
```

## Measuring Privacy Impact

### Topics API Privacy Analysis

Measure the privacy implications of Topics API on your users:

```python
#!/usr/bin/env python3
"""Analyze Topics API privacy implications."""

from collections import Counter
import json

class TopicsPrivacyAnalyzer:
    def __init__(self):
        # Reference list of topic categories
        self.sensitive_topics = [
            'Health/Conditions',
            'Health/Mental Health',
            'Health/Medications',
            'Politics',
            'Religion',
            'LGBTQ+',
            'Financial/Loans',
            'Financial/Credit',
            'Adult',
        ]

    def assess_topic_sensitivity(self, user_topics: list) -> dict:
        """Assess sensitivity of assigned topics."""

        sensitive_matches = [
            t for t in user_topics
            if any(s in t for s in self.sensitive_topics)
        ]

        risk_assessment = {
            'total_topics': len(user_topics),
            'sensitive_topics': sensitive_matches,
            'sensitivity_score': len(sensitive_matches) / max(len(user_topics), 1),
            'risk_level': 'high' if len(sensitive_matches) > 0 else 'low'
        }

        return risk_assessment

    def simulate_user_profiling(self, weekly_topics_history: list) -> dict:
        """Simulate how advertisers could profile users over time."""

        # Flatten history
        all_topics = []
        for week in weekly_topics_history:
            all_topics.extend(week)

        # Find consistent interests
        topic_counts = Counter(all_topics)

        persistent_interests = {
            topic: count for topic, count in topic_counts.items()
            if count > len(weekly_topics_history) * 0.5  # Topic appears in >50% of weeks
        }

        return {
            'consistent_interests': persistent_interests,
            'profiling_risk': len(persistent_interests) / len(topic_counts)
                            if topic_counts else 0
        }

# Usage
analyzer = TopicsPrivacyAnalyzer()

# Simulate week of topics
user_topics_week1 = ['Technology/Programming', 'Health/Mental Health', 'Sports']
assessment = analyzer.assess_topic_sensitivity(user_topics_week1)
print(f"Sensitivity assessment: {json.dumps(assessment, indent=2)}")

# Simulate multiple weeks to see profiling potential
topics_history = [
    ['Technology/Programming', 'Sports/Football', 'News'],
    ['Technology/Programming', 'Finance', 'Sports/Football'],
    ['Technology/Programming', 'Sports/Football', 'Travel'],
]

profiling = analyzer.simulate_user_profiling(topics_history)
print(f"Profiling risk: {json.dumps(profiling, indent=2)}")
```

## Compliance Framework

### GDPR Compliance with Topics API

```yaml
Topics-API-GDPR-Compliance:
  Personal-Data-Assessment:
    Question: "Is a Topics ID personal data?"
    Answer: "Likely yes, as it can be linked to individuals"
    Implication: "Requires legal basis (consent or legitimate interest)"

  Lawful-Basis-Requirements:
    Consent:
      - Must be explicit (opt-in)
      - Must be obtained BEFORE Topics enabled
      - Can be withdrawn anytime

    Legitimate-Interest:
      - Must document balancing test
      - Must show user interest outweighs privacy risk
      - Difficult to establish given sensitivity potential

  User-Rights:
    Right-to-Know: "Users should know what topics assigned"
    Right-to-Object: "Users should be able to opt-out"
    Right-to-Rectify: "Users should verify accuracy"

  Data-Protection-Officer-Assessment:
    - Impact Assessment required
    - Prior consultation with DPA recommended
    - Sensitive topics warrant stronger safeguards
```

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

- [Implement Privacy Preserving Machine Learning](/privacy-tools-guide/how-to-implement-privacy-preserving-machine-learning-for-business-analytics-2026/)
- [EA App Origin Replacement Privacy Data Collection Review.](/privacy-tools-guide/ea-app-origin-replacement-privacy-data-collection-review-ana/)
- [Topics Api Chrome Replacement For Cookies How It Tracks You](/privacy-tools-guide/topics-api-chrome-replacement-for-cookies-how-it-tracks-you/)
- [Android Privacy Indicators: Camera and Mic Access Explained](/privacy-tools-guide/android-privacy-indicators-camera-mic-explained/)
- [Arti Tor Rust Implementation Explained 2026](/privacy-tools-guide/arti-tor-rust-implementation-explained-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
