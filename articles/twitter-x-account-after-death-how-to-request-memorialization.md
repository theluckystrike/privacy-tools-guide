---
layout: default
title: "Twitter X Account After Death: How to Request"
description: "A practical guide for developers and power users on handling Twitter X accounts of deceased users. Learn the memorialization and deactivation processes"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /twitter-x-account-after-death-how-to-request-memorialization/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

When a loved one passes away, their digital footprint requires attention just like their physical belongings. Twitter X (formerly Twitter) provides mechanisms for family members and authorized representatives to request either memorialization or complete removal of deceased user accounts. This guide covers the technical and procedural aspects for developers building tools around this process and power users handling these situations directly.

Prerequisites for Submission

Before submitting a request, gather the following documentation:

- Proof of death (death certificate, obituary, or legal document)
- Your government-issued ID
- The deceased user's account handle or profile URL
- Documentation showing your relationship to the deceased (if not an immediate family member)

Step 2: Submitting the Request

Navigate to Twitter X's Help Center and locate the deceased user account form. The process differs slightly depending on whether you're requesting memorialization or deactivation.

Requesting Account Deactivation

For complete removal, submit through the deceased user deactivation form. Provide:

- The exact Twitter handle (without @ symbol)
- Date of death
- Upload death certificate or equivalent documentation
- Your contact information for follow-up

Requesting Account Memorialization

Memorialization keeps the account visible but prevents modifications. The profile name gains "(Memorial)" suffix, and the account is locked from login. Submit through the memorialization request form with similar documentation.

Step 3: Implement Programmatic Verification for Developers

If you're building an application that tracks account status or automates documentation, you can verify account states through the Twitter API:

```javascript
// Check if an account exists and get basic metadata
const https = require('https');

async function getAccountStatus(username) {
  const cleanUsername = username.replace('@', '');

  // Using Twitter API v2 - requires bearer token
  const bearerToken = process.env.TWITTER_BEARER_TOKEN;

  const options = {
    hostname: 'api.twitter.com',
    path: `/2/users/by/username/${cleanUsername}`,
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${bearerToken}`,
      'Content-Type': 'application/json'
    }
  };

  return new Promise((resolve, reject) => {
    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => {
        if (res.statusCode === 200) {
          const parsed = JSON.parse(data);
          resolve({
            exists: true,
            id: parsed.data.id,
            username: parsed.data.username
          });
        } else if (res.statusCode === 404) {
          resolve({ exists: false });
        } else {
          reject(new Error(`API returned ${res.statusCode}`));
        }
      });
    });
    req.on('error', reject);
    req.end();
  });
}
```

Note that API access requires approval through the Twitter Developer Portal. For deceased user matters, you cannot access the account through the API without proper authorization.

Step 4: Preserving Account Data Before Deactivation

If you need to preserve the deceased user's tweets, you must act quickly. Here's how to document the content:

Using Twitter's Built-in Archive Feature

For your own account or with proper authorization, request a data archive through Settings → Your account → Download your data. The archive includes:

- Tweets and replies
- Direct messages (if enabled)
- Media uploads
- Account activity history

Archiving Public Accounts Programmatically

For public accounts where you cannot obtain direct access, you can archive content using the Twitter API with a rate-limited approach:

```python
import requests
import time
from datetime import datetime
import json

class TwitterArchiver:
    def __init__(self, bearer_token):
        self.bearer_token = bearer_token
        self.base_url = "https://api.twitter.com/2/tweets/search/all"
        self.headers = {
            "Authorization": f"Bearer {bearer_token}",
            "Content-Type": "application/json"
        }

    def archive_user_tweets(self, username, max_results=500):
        """Archive tweets from a user's timeline"""
        all_tweets = []

        # Get user ID first
        user_response = requests.get(
            f"https://api.twitter.com/2/users/by/username/{username}",
            headers=self.headers
        )

        if user_response.status_code != 200:
            raise Exception(f"User not found: {user_response.text}")

        user_id = user_response.json()['data']['id']

        # Fetch tweets with pagination
        next_token = None
        while len(all_tweets) < max_results:
            params = {
                'query': f'from:{username}',
                'max_results': min(100, max_results - len(all_tweets)),
                'tweet.fields': 'created_at,public_metrics,source',
                'sort_order': 'reverse_chronological'
            }

            if next_token:
                params['next_token'] = next_token

            response = requests.get(
                self.base_url,
                params=params,
                headers=self.headers
            )

            if response.status_code != 200:
                raise Exception(f"API error: {response.text}")

            data = response.json()
            if 'data' in data:
                all_tweets.extend(data['data'])

            if 'next_token' in data.get('meta', {}):
                next_token = data['meta']['next_token']
            else:
                break

            # Respect rate limits
            time.sleep(1)

        return all_tweets

    def save_archive(self, username, tweets):
        """Save tweets to a JSON file"""
        filename = f"twitter_archive_{username}_{datetime.now().strftime('%Y%m%d')}.json"

        with open(filename, 'w', encoding='utf-8') as f:
            json.dump({
                'username': username,
                'archive_date': datetime.now().isoformat(),
                'tweet_count': len(tweets),
                'tweets': tweets
            }, f, indent=2, ensure_ascii=False)

        return filename
```

This script respects Twitter's API rate limits and produces a portable JSON archive that can be stored indefinitely.

Step 5: Handling Multiple Accounts

If you're managing digital estate matters for multiple deceased users, consider building a tracking system:

```javascript
// Track multiple deceased user account requests
class DeceasedAccountTracker {
  constructor() {
    this.requests = new Map();
  }

  addRequest(accountHandle, requestType, submittedDate, status = 'pending') {
    this.requests.set(accountHandle, {
      handle: accountHandle,
      type: requestType,
      submitted: submittedDate,
      status: status,
      documents: [],
      notes: []
    });
  }

  updateStatus(accountHandle, status) {
    if (this.requests.has(accountHandle)) {
      const request = this.requests.get(accountHandle);
      request.status = status;
      request.lastUpdated = new Date().toISOString();
    }
  }

  addNote(accountHandle, note) {
    if (this.requests.has(accountHandle)) {
      this.requests.get(accountHandle).notes.push({
        timestamp: new Date().toISOString(),
        content: note
      });
    }
  }

  getReport() {
    const report = [];
    for (const [handle, data] of this.requests) {
      report.push({
        handle,
        type: data.type,
        status: data.status,
        submitted: data.submitted,
        daysSinceSubmission: Math.floor(
          (Date.now() - new Date(data.submitted).getTime()) / (1000 * 60 * 60 * 24)
        )
      });
    }
    return report.sort((a, b) => b.daysSinceSubmission - a.daysSinceSubmission);
  }
}
```

Step 6: What Twitter X Cannot Provide

It's important to understand the limitations of the deceased user process:

- No account access: Family members cannot log into memorialized or deactivated accounts
- No direct message retrieval: Direct messages are not included in data archives for deceased accounts unless the family had prior authorization
- No retroactive changes: Once an account is deactivated, it cannot be recovered
- Limited response times: Twitter X processes these requests manually, which can take several weeks

Step 7: Alternative Considerations

For developers building digital estate tools, consider integrating with multiple platforms:

- Facebook: Legacy contact system
- Instagram: Memorialized accounts (linked to Facebook)
- Google: Inactive Account Manager
- LinkedIn: Family member request process

Each platform has its own policies and timelines. Building a unified tracking system helps manage multiple requests across services.

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to request?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Can I adapt this for a different tech stack?

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Instagram Memorialization Request Process What Happens](/instagram-memorialization-request-process-what-happens-to-ph/)
- [Twitter X Privacy Settings Recommended 2026](/twitter-x-privacy-settings-recommended-2026/)
- [Windows Local Account Vs Microsoft Account](/windows-local-account-vs-microsoft-account-privacy/)
- [LinkedIn Deceased Member Profile Removal How To Report](/linkedin-deceased-member-profile-removal-how-to-report-and-m/)
- [Subscription Service Cancellation After Death](/subscription-service-cancellation-after-death-how-executor-can-close-recurring-payment-accounts-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
