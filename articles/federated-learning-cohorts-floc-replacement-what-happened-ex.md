---
layout: default
title: "Federated Learning Cohorts: FLoC Replacement Explained"
description: "Federated Learning Cohorts: FLoC Replacement Explained — privacy guide covering tools, techniques, and best practices to protect your data and digital"
date: 2026-03-16
author: theluckystrike
permalink: /federated-learning-cohorts-floc-replacement-what-happened-ex/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
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

For developers working with the Privacy Sandbox APIs in 2026, here's the current landscape:

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


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
