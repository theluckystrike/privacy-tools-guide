---
layout: default
title: "Set Up Data Subject Access Request Workflow for GDPR Compliance"
description: "A practical technical guide for developers building GDPR-compliant data subject access request (DSAR) workflows. Covers automated pipelines, identity."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-set-up-data-subject-access-request-workflow-for-gdpr-/
categories: [guides]
intent-checked: true
voice-checked: true
reviewed: true
score: 8
---

{% raw %}
# How to Set Up Data Subject Access Request Workflow for GDPR Compliance

The European Union's General Data Protection Regulation (GDPR) grants individuals significant rights over their personal data. Among the most operationally demanding is the **Right of Access** (Article 15), commonly called a Data Subject Access Request (DSAR). Organizations must respond to these requests within 30 days—and failure can result in fines reaching €20 million or 4% of global annual revenue.

Building a DSAR workflow requires more than a generic contact form. You need identity verification, data discovery across systems, response generation, and audit trails. This guide walks through implementing a practical DSAR pipeline suited for developer and power user skill levels.

## Understanding DSAR Requirements

Before writing code, understand what GDPR actually requires:

- **Scope**: All personal data your organization holds about the requester
- **Format**: Must be provided in a "commonly used electronic form" unless otherwise requested
- **Timeline**: One calendar month from receipt (extendable to two months for complex requests)
- **Verification**: You must confirm the requester's identity before disclosing data

Personal data under GDPR includes not just obvious fields like name and email, but also IP addresses, device identifiers, transaction histories, and any recorded preferences or interactions.

## Designing Your DSAR Pipeline

A DSAR workflow consists of five stages:

1. **Intake**: Receive and log the request
2. **Verification**: Confirm the requester's identity
3. **Discovery**: Locate all personal data across your systems
4. **Compilation**: Assemble the data package
5. **Delivery**: Provide the response within the deadline

### Building the Intake System

Your intake form should capture essential information while setting expectations. Here's a minimal intake endpoint using Express.js:

```javascript
const express = require('express');
const router = express.Router();
const { v4: uuidv4 } = require('uuid');

const dsarRequests = new Map(); // Replace with database in production

router.post('/api/dsar/request', async (req, res) => {
  const { email, fullName, requestType } = req.body;
  
  // Generate unique request ID
  const requestId = uuidv4();
  const timestamp = new Date();
  const deadline = new Date(timestamp);
  deadline.setDate(deadline.getDate() + 30);
  
  const request = {
    id: requestId,
    email,
    fullName,
    requestType: requestType || 'access', // access, erasure, portability, rectification
    status: 'pending_verification',
    receivedAt: timestamp,
    deadline: deadline,
    verificationToken: crypto.randomBytes(32).toString('hex'),
    dataLocations: [] // To be populated during discovery
  };
  
  dsarRequests.set(requestId, request);
  
  // TODO: Send verification email with token
  // TODO: Log to audit system
  
  res.status(202).json({
    requestId,
    message: 'Request received. Please verify your identity.',
    deadline: deadline.toISOString()
  });
});
```

### Implementing Identity Verification

You cannot fulfill a DSAR without confirming the requester's identity. The complexity of verification depends on how much data you have and the sensitivity of your systems. A practical approach includes:

**Step 1: Email Verification**
Send a verification link to the email address on file. This confirms the requester has access to that inbox.

```javascript
async function sendVerificationEmail(request) {
  const verificationUrl = `${process.env.DSAR_PORTAL_URL}/verify/${request.id}/${request.verificationToken}`;
  
  const mailOptions = {
    from: 'privacy@yourcompany.com',
    to: request.email,
    subject: 'Verify your Data Subject Access Request',
    html: `
      <p>We received a request to access your personal data.</p>
      <p>Click <a href="${verificationUrl}">here</a> to verify your identity.</p>
      <p>This link expires in 48 hours.</p>
    `
  };
  
  await transporter.sendMail(mailOptions);
}
```

**Step 2: Knowledge-Based Authentication**
For higher assurance, ask the requester to confirm specific information only they would know—previous addresses, account creation date, or recent transactions.

```javascript
async function verifyIdentityByKBA(request, answers) {
  // Example: Check against stored account metadata
  const user = await getUserByEmail(request.email);
  
  const validAnswers = await Promise.all([
    verifyAnswer(user.accountCreated, answers.accountCreated),
    verifyAnswer(user.previousAddresses, answers.previousAddress),
    verifyAnswer(user.lastTransactionDate, answers.lastTransaction)
  ]);
  
  const confidence = validAnswers.filter(Boolean).length / validAnswers.length;
  return confidence >= 0.75; // Require 75% match
}
```

### Building the Data Discovery Engine

The most challenging part of DSAR compliance is finding all personal data across your infrastructure. Personal data scatters across databases, log files, third-party services, and backup systems.

**Data Mapping**

Before building code, document where personal data lives:

| System | Data Types | Retention | Access Method |
|--------|------------|-----------|---------------|
| User database | Profile, preferences | Duration of account | SQL query |
| Analytics | IP, behavior | 2 years | BigQuery API |
| Support tickets | Communications | 5 years | Zendesk API |
| Email archives | All communications | 7 years | IMAP/SMTP |
| Backups | All of above | 30 days | Restore from S3 |

**Discovery Query Pattern**

```javascript
async function discoverUserData(userEmail, requestId) {
  const results = {
    requestId,
    discoveredAt: new Date().toISOString(),
    sources: []
  };
  
  // Query primary database
  const userData = await db.users.findOne({ email: userEmail });
  results.sources.push({
    system: 'primary_database',
    data: {
      profile: userData,
      // Exclude passwords and tokens
      passwordHash: undefined,
      resetToken: undefined
    }
  });
  
  // Query analytics (with IP anonymization)
  const analyticsEvents = await analytics.query(
    `SELECT event_type, timestamp, 
     GENERATE_UUID() as anonymous_id
     WHERE user_email = @email`,
    { email: userEmail }
  );
  results.sources.push({
    system: 'analytics_platform',
    data: analyticsEvents,
    note: 'IP addresses anonymized per GDPR pseudonymization requirements'
  });
  
  // Query support tickets
  const tickets = await zendesk.search({ type: 'ticket', query: `requester:${userEmail}` });
  results.sources.push({
    system: 'support_platform',
    data: tickets
  });
  
  return results;
}
```

### Compiling the Response Package

Once you've gathered data from all sources, compile it into a portable format. GDPR requires providing data in "commonly used electronic form"—JSON or CSV works well for structured data.

```javascript
async function compileResponse(requestId) {
  const request = await getRequest(requestId);
  const discoveryResults = await discoverUserData(request.email, requestId);
  
  const responsePackage = {
    request: {
      id: request.id,
      type: request.requestType,
      received: request.receivedAt,
      fulfilled: new Date().toISOString()
    },
    personalData: discoveryResults.sources,
    dataCategories: categorizeData(discoveryResults.sources),
    processingActivities: getProcessingActivities(request.email),
    thirdPartySharing: getThirdPartySharing(request.email),
    rightsExplanation: {
      rectification: 'You may request correction of inaccurate data',
      erasure: 'You may request deletion in certain circumstances',
      portability: 'You may receive data in machine-readable format',
      objection: 'You may object to processing based on legitimate interests'
    }
  };
  
  // Generate downloadable package
  const packagePath = await generateDataPackage(responsePackage);
  
  await updateRequestStatus(requestId, 'completed', { packagePath });
  
  return packagePath;
}
```

### Tracking and Deadlines

Missing the 30-day deadline is one of the most common compliance failures. Implement automatic deadline tracking:

```javascript
async function checkAndAlertDeadlines() {
  const thirtyDaysFromNow = new Date();
  thirtyDaysFromNow.setDate(thirtyDaysFromNow.getDate() + 30);
  
  const approachingDeadlines = await db.dsarRequests.find({
    status: { $ne: 'completed' },
    deadline: { $lte: thirtyDaysFromNow }
  });
  
  for (const request of approachingDeadlines) {
    const daysRemaining = Math.ceil(
      (request.deadline - new Date()) / (1000 * 60 * 60 * 24)
    );
    
    if (daysRemaining <= 7) {
      // Send urgent alert
      await sendAlert({
        to: 'compliance-team@yourcompany.com',
        subject: `URGENT: DSAR deadline in ${daysRemaining} days`,
        requestId: request.id,
        requester: request.email
      });
    }
  }
}
```

## Handling Erasure Requests

The Right to Erasure (Article 17) adds complexity—data must be deleted not just from your primary systems but from backups, third-party processors, and any cached copies.

A practical pattern marks data for deletion with a grace period, then removes it across systems:

```javascript
async function processErasureRequest(requestId) {
  const request = await getRequest(requestId);
  
  // Verify identity before processing
  if (!await verifyIdentity(request)) {
    throw new Error('Identity verification failed');
  }
  
  // Mark for deletion (with 30-day grace period for legal holds)
  await markDataForDeletion(request.email, {
    immediate: ['marketing_preferences', 'session_data'],
    delayed: ['user_profile', 'transaction_history'],
    exclude: ['financial_records', 'legal_compliance']
  });
  
  // Schedule actual deletion
  const deletionDate = new Date();
  deletionDate.setDate(deletionDate.getDate() + 30);
  
  await scheduleDeletionJob({
    email: request.email,
    executeAt: deletionDate,
    systems: ['primary_db', 'analytics', 'backup_s3']
  });
  
  await updateRequestStatus(requestId, 'erasure_scheduled', {
    deletionDate: deletionDate.toISOString()
  });
}
```

## Automating Where Possible

Manual DSAR handling doesn't scale. As request volume grows, automate:

- **Intake**: Self-service portal with email verification
- **Discovery**: Scheduled queries across connected systems
- **Alerts**: Automatic escalation as deadlines approach
- **Compilation**: Template-based response generation

However, retain human oversight for complex cases—ambiguous requests, potential legal holds, or requests affecting multiple jurisdictions.

## Testing Your Workflow

Before relying on your DSAR pipeline, validate it works:

1. **Submit test requests** through your production portal using a test account
2. **Verify complete data discovery** by comparing against known data in your systems
3. **Test deadline handling** by backdating requests and confirming alerts fire
4. **Validate erasure** by attempting to access deleted data across all systems
5. **Audit trail verification** ensuring all actions are logged with timestamps

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Troubleshooting Hub](/privacy-tools-guide/troubleshooting-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
