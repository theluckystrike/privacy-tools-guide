---
layout: default
title: "Use Email Subaddressing Plus Addressing For Tracking Which Services Leak Your Address"
description: "A practical guide to using email subaddressing and addressing techniques to track which services leak your email address. Perfect for developers"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-use-email-subaddressing-plus-addressing-for-tracking-which-services-leak-your-address/
categories: [guides, security]
reviewed: true
intent-checked: true
voice-checked: true
score: 8
tags: [privacy-tools-guide]
---

{% raw %}

Email subaddressing (also known as plus addressing or tagged addressing) is a powerful technique that lets you create infinite email variations using a single inbox. When combined with strategic addressing patterns, you can not only organize your email but also detect which services leak or sell your personal information. This guide covers practical implementations for developers and power users who want to maintain control over their inbox and track data handling practices.

## Understanding Email Subaddressing

Most email providers support a simple syntax: anything after a plus sign (`+`) in your email address gets ignored during delivery but gets preserved in the recipient metadata. If your primary email is `developer@example.com`, you can use `developer+github@example.com` for GitHub registrations. Both addresses deliver to the same inbox, but the recipient sees the full address.

This RFC 5233 compliant feature works with Gmail, Outlook, iCloud, Proton Mail, and most modern email services. The key advantage is that you can create unique addresses for every service without managing multiple accounts.

## Setting Up Subaddressing for Tracking

Create a systematic approach to tracking email usage across services. Generate aliases using a consistent format that identifies the service name:

```javascript
// Generate subaddresses for service tracking
function createServiceEmail(serviceName, baseEmail = 'developer@example.com') {
  const [local, domain] = baseEmail.split('@');
  return `${local}+${serviceName.toLowerCase().replace(/\s+/g, '-')}@${domain}`;
}

// Usage examples
const serviceEmails = {
  github: createServiceEmail('github'),
  aws: createServiceEmail('aws'),
  linkedin: createServiceEmail('linkedin'),
  newsletter: createServiceEmail('newsletter-weekly'),
  ecommerce: createServiceEmail('shop-amazon')
};

console.log(serviceEmails.github);   // developer+github@example.com
console.log(serviceEmails.aws);      // developer+aws@example.com
```

This script generates predictable aliases that you can document in a password manager or dedicated tracking system. When you sign up for a new service, create a subaddress that clearly identifies the service name.

## Detecting Address Leaks

Once you've been using subaddresses for a while, detecting leaks becomes straightforward. Create a dedicated email filter that labels incoming mail by the alias used:

```bash
# Gmail filter example - create filters for each service
# Match: to:(developer+github@example.com)
# Action: Add label "service/github", never mark as spam
```

When `developer+github@example.com` starts receiving spam from unrelated domains, you know GitHub either experienced a breach or shared your data with third parties. Similarly, if your `linkedin+newsletter@example.com` address suddenly receives promotional emails about insurance, LinkedIn's data sharing practices are now affecting your inbox.

## Advanced: Custom Domain Subaddressing

For maximum control, pair subaddressing with a custom domain. This approach provides additional benefits:

1. **Portability**: Change email providers without updating addresses
2. **Flexibility**: Create unlimited aliases without provider limits
3. **Branding**: Use professional domain names

```bash
# Cloudflare Email Routing (free) setup example
# 1. Add your domain to Cloudflare
# 2. Enable Email Routing
# 3. Configure catch-all to forward to your primary inbox
# 4. Start using aliases like:
#    - dev@yourdomain.com
#    - dev+shop@yourdomain.com  
#    - dev+travel@yourdomain.com
```

Services like Cloudflare Email Routing, Forward Email, and ImprovMX offer free catch-all forwarding. Combined with a privacy-respecting inbox provider, you gain full control over incoming mail.

## Identifying Service Data Breaches

When a service you registered with experiences a breach, attackers often obtain email addresses. If you're using subaddresses, you can identify the compromised service immediately:

1. **Monitor incoming mail** to subaddresses for spam patterns
2. **Track unsubscribe behavior** - legitimate services honor unsubscribe requests
3. **Note reply patterns** - some services send automated replies to verify email validity

Create a simple logging system:

```javascript
// Track which addresses receive mail
const emailLog = [];

function logIncomingEmail(fullAddress) {
  const match = fullAddress.match(/^([^@]+)\+([^@]+)@(.+)$/);
  if (match) {
    const [, user, service, domain] = match;
    emailLog.push({
      timestamp: new Date().toISOString(),
      service: service,
      address: fullAddress
    });
    console.log(`Email received for service: ${service}`);
  }
}

// Later: analyze emailLog to detect anomalies
function detectLeak(serviceName) {
  const relevantLogs = emailLog.filter(e => e.service === serviceName);
  const uniqueSenders = new Set(relevantLogs.map(e => extractDomain(e.from)));
  console.log(`${serviceName} has been contacted by ${uniqueSenders.size} unique senders`);
}
```

## Practical Implementation Strategy

Start with these steps to build your tracking system:

1. **Choose your base email** - use a privacy-respecting provider
2. **Adopt a naming convention** - `service-purpose-date` format works well
3. **Create a tracking document** - maintain a list of all active aliases
4. **Set up filters** - automatically label incoming mail by service
5. **Review quarterly** - check for unexpected senders or patterns

When you notice a leak, you have options:
- Block the specific alias entirely
- Contact the service to request data removal
- Use the alias for targeted spam filtering in your email client

## Common Pitfalls to Avoid

Some services don't accept plus addresses due to poor validation. Workarounds include:

- Using dots in Gmail (`d.e.v.e.l.o.p.e.r@example.com` is identical to `developer@example.com` but appears unique)
- Creating actual aliases in your email provider settings
- Using catch-all domain forwarding for maximum flexibility

Avoid using easily guessable patterns like sequential numbers (`user+1@example.com`, `user+2@example.com`) as these can be trivially stripped by spam systems.

## Automating Alias Management

For power users managing many services, automate the process:

```javascript
// Automated alias generator with logging
class EmailAliasManager {
  constructor(baseEmail, storage) {
    this.baseEmail = baseEmail;
    this.aliases = new Map();
    this.storage = storage;
  }

  generate(service, purpose = 'general') {
    const key = `${service}-${purpose}`;
    if (!this.aliases.has(key)) {
      const alias = this.createAlias(service, purpose);
      this.aliases.set(key, alias);
      this.storage.save(key, alias);
    }
    return this.aliases.get(key);
  }

  createAlias(service, purpose) {
    const [local, domain] = this.baseEmail.split('@');
    const timestamp = Date.now().toString(36);
    return `${local}+${service}-${purpose}-${timestamp}@${domain}`;
  }

  lookup(alias) {
    return [...this.aliases.entries()].find(([_, a]) => a === alias);
  }
}
```

This approach creates discoverable aliases that include service name, purpose, and a timestamp for uniqueness while remaining readable.



## Related Reading

- [WebRTC Local IP Leak: How It Reveals Your Real Address](/privacy-tools-guide/webrtc-local-ip-leak-how-it-reveals-your-real-address/)
- [How To Detect If Your Email Address Has Been Sold To Marketi](/privacy-tools-guide/how-to-detect-if-your-email-address-has-been-sold-to-marketi/)
- [How To Set Up Forwarding Only Email Address That Hides Your](/privacy-tools-guide/how-to-set-up-forwarding-only-email-address-that-hides-your-/)
- [Privacy-Focused Email Forwarding Services Comparison](/privacy-tools-guide/privacy-focused-email-forwarding-services-comparison/)
- [Someone Signed Up for Services Using My Email — What to Do](/privacy-tools-guide/someone-signed-up-for-services-using-my-email-what-to-do/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
