---
layout: default
title: "How to Build a Privacy-Preserving Webhook Relay That Strips PII Before Delivery"
description: "Learn how to build a webhook relay system that automatically strips personally identifiable information (PII) before forwarding webhooks to your endpoints."
date: 2026-03-16
author: "Privacy Tools Guide"
permalink: /how-to-build-privacy-preserving-webhook-relay-that-strips-pi/
reviewed: true
score: 8
categories: [guides]
---
{% raw %}
Webhooks are the backbone of modern event-driven architectures. They power everything from payment notifications to CI/CD pipelines. But handling webhooks from third-party services often means receiving sensitive user data you don't need—and shouldn't store. Building a privacy-preserving webhook relay gives you control over what data reaches your systems.

This guide walks you through creating a webhook relay that strips PII before delivery, keeping your systems compliant and your data minimal.

## Why Strip PII from Webhooks?

When Stripe sends you a payment webhook, it includes customer email addresses, names, and billing details. When a SaaS platform notifies you of a new user signup, it might include full names and phone numbers. If your system only needs the event type and ID, you're accumulating liability by storing unnecessary PII.

A privacy-preserving relay intercepts incoming webhooks, removes or redacts sensitive fields, and forwards only what your system requires. This approach reduces your compliance burden, minimizes data breach exposure, and enforces data minimization principles.

## Architecture Overview

The relay operates as middleware between webhook providers and your endpoints. It receives POST requests, applies transformation rules, and forwards the cleaned payload to your destination.

```
Webhook Provider → Relay (PII Stripping) → Your Endpoint
```

The relay needs three core components:
- An HTTP server to receive webhooks
- A rule engine to define which fields to strip
- A forwarding mechanism to send transformed payloads

## Building the Relay in Node.js

Here's a complete implementation using Express:

```javascript
const express = require('express');
const axios = require('axios');

const app = express();
app.use(express.json());

// Define PII field patterns to strip
const PII_PATTERNS = [
  'email', 'phone', 'first_name', 'last_name', 'full_name',
  'ssn', 'credit_card', 'billing_address', 'ip_address'
];

function stripPII(obj, path = '') {
  if (typeof obj !== 'object' || obj === null) {
    return obj;
  }

  const result = Array.isArray(obj) ? [] : {};

  for (const [key, value] of Object.entries(obj)) {
    const currentPath = path ? `${path}.${key}` : key;
    const lowerKey = key.toLowerCase();

    // Check if this field matches any PII pattern
    if (PII_PATTERNS.some(pattern => lowerKey.includes(pattern))) {
      result[key] = '[REDACTED]';
    } else if (typeof value === 'object') {
      result[key] = stripPII(value, currentPath);
    } else {
      result[key] = value;
    }
  }

  return result;
}

app.post('/relay', async (req, res) => {
  const { destination, payload } = req.body;

  if (!destination || !payload) {
    return res.status(400).json({ error: 'Missing destination or payload' });
  }

  // Strip PII from the payload
  const sanitizedPayload = stripPII(payload);

  try {
    await axios.post(destination, sanitizedPayload);
    res.json({ status: 'forwarded', sanitized: true });
  } catch (error) {
    res.status(500).json({ error: 'Forwarding failed', details: error.message });
  }
});

app.listen(3000, () => {
  console.log('Privacy-preserving webhook relay running on port 3000');
});
```

## Configuration Options for Power Users

The basic implementation works, but you'll want more control. Extend the relay with configurable rules:

```javascript
const RULES = {
  stripe: {
    strip: ['customer.email', 'customer.phone', 'billing_details'],
    keep: ['id', 'object', 'amount', 'currency', 'status']
  },
  github: {
    strip: ['sender.login', 'sender.avatar_url'],
    keep: ['action', 'repository', 'ref']
  }
};

function applyRules(payload, provider) {
  const rules = RULES[provider];
  if (!rules) {
    return stripPII(payload); // Default: strip all known PII
  }

  const result = {};
  
  // Only keep specified fields
  if (rules.keep) {
    for (const field of rules.keep) {
      const value = getNestedValue(payload, field);
      if (value !== undefined) {
        setNestedValue(result, field, value);
      }
    }
  }

  // Strip explicitly banned fields
  if (rules.strip) {
    for (const field of rules.strip) {
      setNestedValue(result, field, '[REDACTED]');
    }
  }

  return result;
}

// Helper functions for nested object access
function getNestedValue(obj, path) {
  return path.split('.').reduce((current, key) => current?.[key], obj);
}

function setNestedValue(obj, path, value) {
  const keys = path.split('.');
  const last = keys.pop();
  const target = keys.reduce((current, key) => {
    if (!current[key]) current[key] = {};
    return current[key];
  }, obj);
  target[last] = value;
}
```

## Handling Edge Cases

Real-world webhooks have nested structures and arrays. The implementation above handles recursion, but you need to consider a few scenarios.

**Nested arrays containing objects:**
```javascript
// If your webhook has arrays of user objects
const SANITIZE_ARRAYS = true; // Enable recursive array sanitization
```

**Conditional redaction:**
```javascript
// Only redact emails for non-admin users
function smartStrip(obj, context) {
  if (context.isAdmin) {
    return obj; // Don't strip for admins
  }
  return stripPII(obj);
}
```

**Logging without PII:**
```javascript
app.post('/relay', (req, res) => {
  const { destination, payload } = req.body;
  
  // Log only metadata, never the payload
  logger.info('Webhook received', {
    destination,
    provider: req.headers['x-webhook-source'],
    timestamp: new Date().toISOString(),
    payloadSize: JSON.stringify(payload).length
  });
  
  // Then proceed with sanitization
});
```

## Deployment Considerations

Run the relay as a separate service with its own authentication. Don't expose it directly to the internet—your webhook providers should send to your relay, which then forwards internally.

Add authentication:
```javascript
app.post('/relay', (req, res, next) => {
  const token = req.headers['x-api-key'];
  if (token !== process.env.RELAY_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}, async (req, res) => {
  // Relay logic here
});
```

Set up health checks and dead-letter queues for failed forwards:
```javascript
const deadLetterQueue = [];

app.post('/relay', async (req, res) => {
  try {
    await axios.post(destination, sanitizedPayload);
  } catch (error) {
    deadLetterQueue.push({ destination, payload: sanitizedPayload, error: error.message });
    res.status(202).json({ status: 'queued', reason: 'forwarding failed' });
  }
});
```

## Summary

A privacy-preserving webhook relay gives you granular control over the data flowing through your systems. By stripping PII before delivery, you reduce compliance requirements, minimize breach risk, and follow data minimization principles.

The implementation above provides a foundation. Extend it with provider-specific rules, add detailed logging, and deploy with proper authentication. Your endpoints receive only the data they need—nothing more.



## Frequently Asked Questions


**How long does it take to build a privacy-preserving webhook relay that strips pii before delivery?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.


**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.


**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.


**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.


**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.


## Related Articles

- [Best Accessible Privacy Extension for Firefox That Does Not](/best-accessible-privacy-extension-for-firefox-that-does-not-/)
- [Bumble Beeline Data Privacy Who Can See That You Swiped](/bumble-beeline-data-privacy-who-can-see-that-you-swiped-righ/)
- [How To Build Privacy Dashboard For Customers To Manage](/how-to-build-privacy-dashboard-for-customers-to-manage-their/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
