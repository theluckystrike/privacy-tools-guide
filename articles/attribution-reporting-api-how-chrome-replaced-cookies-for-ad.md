---
layout: default
title: "Attribution Reporting API: How Chrome Replaced Cookies."
description: "Learn how Chrome's Attribution Reporting API enables privacy-preserving ad measurement without third-party cookies. Includes code examples and implementation guide."
date: 2026-03-16
author: theluckystrike
permalink: /attribution-reporting-api-how-chrome-replaced-cookies-for-ad/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

The digital advertising industry faces a fundamental shift as third-party cookies phase out across major browsers. Google Chrome's Attribution Reporting API offers a privacy-preserving alternative that allows advertisers to measure campaign effectiveness while reducing cross-site tracking. This guide explains how the API works and provides practical implementation examples for developers.

## Understanding the Attribution Reporting API

The Attribution Reporting API, developed as part of the Privacy Sandbox initiative, enables measurement of conversion events (like purchases or sign-ups) attributed to ad interactions without exposing individual user data. Unlike traditional cookie-based tracking that follows users across websites, this API uses browser-level aggregation to report only summarized data.

The core concept involves three key entities: the site where users see ads (publisher), the site where conversions happen (advertiser), and the browser that coordinates reporting. When a user clicks or views an ad, the browser registers an attribution source. Later, when the user completes a conversion on the advertiser's site, the browser matches the conversion to the source and generates an aggregate report.

## How the API Differs from Cookie-Based Tracking

Traditional third-party cookies allow advertisers to track users across multiple sites, building detailed profiles of user behavior. The Attribution Reporting API fundamentally changes this model by keeping measurement data within the user's browser rather than sharing it across networks.

When you implement the API, the browser handles all matching logic locally. Neither the advertiser nor any third party sees raw user-level data. Instead, reports contain aggregated metrics with noise added to protect individual privacy. This approach satisfies growing regulatory requirements like GDPR and CCPA while still providing useful measurement data.

The API supports two reporting modes: event-level reports and aggregate reports. Event-level reports provide detailed information about specific conversions but include delays and randomization. Aggregate reports offer more accurate data but require additional setup using the Aggregation Service.

## Implementing the Attribution Reporting API

### Step 1: Registering Attribution Sources

First, your ad serving code must register when users interact with advertisements. Add the appropriate headers to your ad impressions:

```http
Attribution-Reporting-Register-Source: {
  "destination": "https://advertiser.example.com",
  "expiry": "604800",
  "source_event_id": "12345678"
}
```

The destination field specifies where conversions can occur. The expiry determines how long after the initial interaction conversions remain attributable. The source_event_id lets you associate a custom identifier with the impression.

In JavaScript, you can register sources dynamically:

```javascript
function registerAdImpression(adData) {
  const registerSource = {
    destination: adData.conversionDestination,
    expiry: 604800, // 7 days in seconds
    source_event_id: adData.campaignId + '_' + adData.adId
  };

  if (window.AttributionReporting) {
    AttributionReporting.registerSource(registerSource);
  }
}
```

### Step 2: Handling Conversions

When users complete desired actions on your landing page, trigger conversion registration:

```http
Attribution-Reporting-Register-Trigger: {
  "triggers": [
    {
      "destination": "https://advertiser.example.com",
      "trigger_data": "purchase_complete",
      "event_report_window": "604800",
      "priority": "1"
    }
  ]
}
```

The trigger_data field lets you categorize conversion types, useful for distinguishing between newsletter signups, purchases, or other actions. Event report windows control when the final report generates.

JavaScript registration works similarly:

```javascript
function registerConversion(conversionType) {
  const registerTrigger = {
    triggers: [{
      destination: window.location.origin,
      trigger_data: conversionType,
      event_report_window: 604800
    }]
  };

  if (window.AttributionReporting) {
    AttributionReporting.registerTrigger(registerTrigger);
  }
}

// Call when purchase completes
registerConversion('purchase_complete');
```

### Step 3: Receiving Reports

Configure your server to receive attribution reports. The browser sends POST requests to your specified endpoints:

```javascript
// Express.js example for receiving event-level reports
app.post('/attribution-reporting/report', express.json(), (req, res) => {
  const report = req.body;
  
  console.log('Attribution Report:', {
    attributionDestination: report.attribution_destination,
    reportTime: report.report_time,
    sourceEventId: report.source_event_id,
    triggerData: report.trigger_data
  });
  
  // Process and store report data
  res.status(200).send('OK');
});
```

Reports arrive with a randomized delay (typically 1-48 hours) to prevent timing-based correlation attacks.

## Aggregate Reporting for Better Accuracy

For more accurate measurement, implement aggregate reporting using the Aggregation Service. This requires additional infrastructure but provides noise-reduced results.

First, define aggregatable keys in your source registration:

```javascript
const registerSource = {
  destination: "https://advertiser.example.com",
  aggregatable_keys: {
    "campaign_counts": "campaign_123",
    "value_sum": "conversion_value"
  }
};
```

Define trigger values when conversions occur:

```javascript
const registerTrigger = {
  triggers: [{
    aggregatable_trigger_data: [
      {
        key_piece: "campaign_123",
        source_keys: ["campaign_counts"]
      },
      {
        key_piece: "conversion_value",
        source_keys: ["value_sum"]
      }
    ],
    value: 1500  // Conversion value in minor units
  }]
};
```

The Aggregation Service then processes these reports into meaningful metrics while applying differential privacy techniques.

## Privacy Budget and Limitations

The API includes privacy budgets that limit how much information any single user can expose. Each source triggers limits the number of reports generated, and noise injection ensures individual conversions cannot be precisely identified.

Understanding these limitations helps set realistic expectations. You cannot track individual user journeys in detail, nor can you achieve pixel-perfect attribution. Instead, focus on aggregate trends and statistical patterns that remain useful for optimization decisions.

## Migrating from Cookie-Based Tracking

If you currently rely on third-party cookies, transition gradually:

1. Implement the Attribution Reporting API alongside existing tracking during a parallel period
2. Validate data consistency between systems before removing legacy code
3. Adjust reporting expectations to account for the API's privacy protections
4. Train your analytics team on interpreting aggregate data versus individual user paths

Many advertising platforms now support this API natively, so check whether your existing tools have built-in integrations before building custom implementations.

## Browser Support and Considerations

The Attribution Reporting API ships in Chrome, Edge, and other Chromium-based browsers. Safari and Firefox implement their own privacy-preserving alternatives with different APIs. For comprehensive measurement across browsers, consider using multiple measurement approaches or relying on server-side tracking with appropriate consent mechanisms.

Always implement proper consent interfaces before collecting any measurement data. Regulatory requirements vary by jurisdiction, and the API itself provides no consent management.

## Getting Started

Begin by auditing your current conversion tracking implementation. Identify the key events you measure and the decisions those measurements inform. Then implement the Attribution Reporting API for at least one conversion type, comparing results against your existing setup during a testing phase.

The transition away from third-party cookies represents both a challenge and an opportunity to build more privacy-respecting measurement systems. The Attribution Reporting API provides the technical foundation for continuing to understand advertising effectiveness while respecting user privacy.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
