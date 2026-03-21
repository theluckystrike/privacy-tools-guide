---
layout: default
title: "GDPR Data Subject Access Request Template"
description: "Learn how to create and process GDPR data subject access requests. Includes practical templates, code examples, and automation strategies for developers"
date: 2026-03-15
author: theluckystrike
permalink: /gdpr-data-subject-access-request-template/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# GDPR Data Subject Access Request Template

Use the DSAR response template below to handle GDPR Article 15 data subject access requests within the required one-month deadline. It includes a ready-to-copy markdown response template, a Python handler class that automates data collection and response generation, an Express.js route for accepting requests via API, and a data inventory schema for discovering all user data across your systems. Adapt these components to your stack to build a compliant, automated DSAR workflow.

## Understanding the DSAR Requirement

When an individual submits a DSAR, your organization must respond within one month. This deadline can extend to two months for complex requests, but you must inform the requester within the first month. The response must include:

- Confirmation of whether personal data is being processed
- Access to the personal data itself
- Information about processing purposes, data categories, and recipients
- The planned retention period or criteria for determining it
- The right to rectification, erasure, or restriction of processing
- The right to lodge a complaint with a supervisory authority
- The source of the data if not collected directly from the subject

## DSAR Template for Data Controllers

Use this template as a starting point for your DSAR handling process:

```markdown
# Data Subject Access Request Response

Request ID: {{request_id}}
Date Received: {{received_date}}
Requester: {{requester_name}}
Email: {{requester_email}}

## Personal Data Found

{% for data_category in personal_data %}
### {{data_category.category_name}}
- Data: {{data_category.content}}
- Source: {{data_category.source}}
- Retention: {{data_category.retention_period}}
{% endfor %}

## Processing Activities

| Purpose | Legal Basis | Categories | Recipients |
|---------|-------------|------------|------------|
{{#each processing_activities}}
| {{this.purpose}} | {{this.legal_basis}} | {{this.categories}} | {{this.recipients}} |
{{/each}}

## Rights Summary

- Right to rectification: You may request correction of inaccurate data
- Right to erasure: Under certain conditions, you may request deletion
- Right to restriction: You may request processing limitation
- Right to data portability: Your data can be exported in machine-readable format
```

## Implementing DSAR Handling in Code

For developers, automating DSAR handling reduces manual effort and ensures consistency. Here are practical implementations:

### Python DSAR Request Handler

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional
import uuid

@dataclass
class DSARRequest:
    request_id: str
    requester_email: str
    requester_name: str
    request_date: datetime
    deadline: datetime
    status: str = "pending"
    verified: bool = False

class DSARHandler:
    def __init__(self, data_store, identity_verifier):
        self.data_store = data_store
        self.identity_verifier = identity_verifier
        self.gdpr_retention_days = 30
    
    def create_request(self, email: str, name: str) -> DSARRequest:
        """Create a new DSAR request with statutory deadline."""
        request_id = str(uuid.uuid4())[:8]
        request_date = datetime.utcnow()
        
        # GDPR mandates one month response time
        deadline = request_date + timedelta(days=30)
        
        return DSARRequest(
            request_id=request_id,
            requester_email=email,
            requester_name=name,
            request_date=request_date,
            deadline=deadline
        )
    
    def collect_user_data(self, user_id: str) -> dict:
        """Collect all personal data associated with a user."""
        user_data = self.data_store.get_user(user_id)
        
        return {
            "profile": user_data.profile,
            "activity_logs": self.data_store.get_activity(user_id),
            "communications": self.data_store.get_communications(user_id),
            "payments": self.data_store.get_payments(user_id),
            "analytics": self.data_store.get_analytics_events(user_id)
        }
    
    def generate_response(self, dsar_request: DSARRequest) -> dict:
        """Generate a DSAR response with all required information."""
        user_data = self.collect_user_data(dsar_request.requester_email)
        processing_activities = self.data_store.get_processing_activities(
            dsar_request.requester_email
        )
        
        return {
            "request_id": dsar_request.request_id,
            "response_date": datetime.utcnow().isoformat(),
            "personal_data": user_data,
            "processing_activities": processing_activities,
            "data_categories": self._categorize_data(user_data),
            "retention_info": self._get_retention_info(user_data)
        }
```

### Express.js Route for DSAR Submission

```javascript
const express = require('express');
const router = express.Router();
const { body, validationResult } = require('express-validator');

const dsarValidation = [
    body('email').isEmail().normalizeEmail(),
    body('fullName').trim().isLength({ min: 2 }),
    body('requestType').isIn(['access', 'erasure', 'portability', 'rectification'])
];

router.post('/api/dsar/submit', dsarValidation, async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(400).json({ errors: errors.array() });
    }
    
    const { email, fullName, requestType } = req.body;
    
    // Verify identity before processing
    const identityVerified = await identityService.verifyIdentity(email, fullName);
    
    if (!identityVerified) {
        return res.status(403).json({
            error: 'Identity verification required',
            message: 'Please provide additional verification information'
        });
    }
    
    // Create and store DSAR request
    const request = await dsarService.createRequest({
        email,
        fullName,
        type: requestType,
        submittedAt: new Date()
    });
    
    res.status(202).json({
        requestId: request.id,
        message: 'DSAR request received',
        deadline: request.deadline.toISOString()
    });
});
```

## Automating Data Discovery

One of the hardest parts of DSAR handling is discovering all places where user data exists. A systematic approach:

### Data Inventory Schema

Maintain a data inventory that tracks where personal data lives:

```yaml
# data-inventory.yaml
data_stores:
  - name: user_profiles
    type: PostgreSQL
    contains_pii: true
    pii_fields:
      - email
      - full_name
      - phone_number
      - ip_address
    retention_policy: "7 years after deletion"
    
  - name: session_logs
    type: Elasticsearch
    contains_pii: true
    pii_fields:
      - ip_address
      - user_agent
    retention_policy: "90 days"
    
  - name: email_queue
    type: RabbitMQ
    contains_pii: true
    pii_fields:
      - recipient_email
    retention_policy: "30 days after delivery"
```

### Discovery Query Pattern

```sql
-- Example: Finding all user data across tables
SELECT 'users' as source, id, email, created_at FROM users WHERE email = ?
UNION ALL
SELECT 'profiles', user_id, NULL, created_at FROM profiles WHERE user_id IN (SELECT id FROM users WHERE email = ?)
UNION ALL
SELECT 'activity_log', user_id, NULL, created_at FROM activity_log WHERE user_id IN (SELECT id FROM users WHERE email = ?);
```

## Best Practices for DSAR Handling

DSARs are a common attack vector, so require multi-factor verification before processing any request. The statutory clock starts at receipt, not at verification, so log the incoming request immediately.

Maintain audit trails of what data you found and what you provided. If you cannot fulfill all aspects of a request, document what you cannot provide and the legal reason why.

Manual DSAR handling is error-prone and does not scale — automate data collection and response generation wherever the architecture permits. Ensure everyone on your team understands the legal requirements and your internal processes.

## Making DSAR Handling Sustainable

Treating DSAR handling as part of your data architecture — not a compliance afterthought — is what makes it sustainable. Build templates, automate data discovery, and follow systematic processes to meet GDPR requirements without degrading operational efficiency.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [How to Set Up Data Subject Access Request Workflow for.](/privacy-tools-guide/how-to-set-up-data-subject-access-request-workflow-for-gdpr-/)
- [GDPR Data Processing Agreement Template Guide: A Developer's Practical Reference](/privacy-tools-guide/gdpr-data-processing-agreement-template-guide/)
- [GDPR Compliant Email Marketing Guide 2026: A Developer](/privacy-tools-guide/gdpr-compliant-email-marketing-guide-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
