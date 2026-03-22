---
layout: default
title: "Facebook Privacy Settings 2026 Complete Guide"
description: "Facebook Privacy Settings 2026: A Complete Guide for. — privacy guide covering tools, techniques, and best practices to protect your data and digital"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /facebook-privacy-settings-2026-complete-guide/
voice-checked: true
reviewed: true
score: 9
categories: [guides]
intent-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

Control Facebook privacy through Settings > Privacy by restricting "Who can see your posts" to Friends only, disabling platform tracking in Settings > Apps and Websites, using "Clear History" to remove apps' data access, and setting Location Services to "Off." For developers, restrict API access through Facebook Developer settings, requiring user re-authentication before accessing sensitive data. Configure audience defaults to "Friends" instead of "Public," disable facial recognition in privacy settings, and review connected apps monthly to revoke unnecessary data access.

## Key Takeaways

- **Running "Limit past posts"**: takes 30 seconds and blocks all pre-2024 public posts.
- **Who can post on your timeline?**: Set to "Friends" or "Only me" to prevent unauthorized posts
2.
- **This tool is particularly**: useful for reviewing post audience defaults, checking profile visibility, adjusting tagging controls, and reviewing app permissions.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Consider a security review**: if your application handles sensitive user data.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Access Your Privacy Dashboard

Navigate to **Settings & Privacy → Settings → Privacy** to access the central hub for all privacy controls. The interface organizes settings into three categories: **Who can see your content**, **Who can contact you**, and **Your Facebook information**.

For developers building applications that interact with Facebook data, understanding these controls is essential because user-configured restrictions directly impact what your application can access through the API.

### Step 2: Audience Selection Fundamentals

The audience selector appears throughout Facebook, but the default settings determine baseline visibility. Configure these in the "Who can see your content" section:

- **Public**: Anyone on or off Facebook
- **Friends**: Your Facebook connections only
- **Friends except...**: All friends except specified individuals
- **Only me**: Visible solely to your account
- **Custom**: Specific people or lists

To set defaults for future posts, use the "Limit who can see past posts" tool, which batch-updates all previous Public and Friends-of-Friends posts to Friends-only visibility.

### Step 3: Profile Privacy for Developers

When building applications that request profile data, be aware that the following fields require explicit user permission:

```javascript
// Required permissions for profile access
const requiredPermissions = [
  'public_profile',
  'email',
  'user_friends',
  'user_photos',
  'user_posts'
];

// Check current permissions before making API calls
FB.api('/me/permissions', 'GET', (response) => {
  console.log('Current permissions:', response.data);
});
```

Users can review granted permissions at any time by visiting **Settings → Apps and Websites**. Revoking permissions immediately restricts API access for that application.

### Step 4: Control Tagging and Timeline

The "Timeline and Tagging" settings determine how others interact with your profile:

1. **Who can post on your timeline?** — Set to "Friends" or "Only me" to prevent unauthorized posts
2. **Who can see what others post on your timeline?** — Controls visibility of content others tag you in
3. **Review posts you're tagged in before the post appears on your timeline** — Enables manual approval queue

For developers, the `user_tagged_places` and `user_photos` permissions trigger these review settings. Applications should handle cases where content is pending review or rejected.

### Step 5: Story and Reel Privacy

Facebook's "Stories" and "Reels" have separate privacy controls from standard posts. Access these under **Settings → Privacy → Story settings**:

- **Who can see your story** — Independent audience selection
- **Allow others to share your story as messages** — Prevents forwarding
- **Hide story from** — Exclude specific users

### Step 6: Data Download and Portability

Facebook provides data export options under **Settings → Your Facebook Information → Download your information**. You can request:

- **All available data** or custom date ranges
- Specific data categories: posts, photos, messages, ads interactions
- Format options: JSON or HTML archive

For developers, the Data Download tool is useful for auditing what data Facebook stores. The exported JSON includes metadata that reveals how Facebook categorizes your activity:

```json
{
  "profile_update": "2026-01-15T10:30:00+0000",
  "ad_interactions": [
    {
      "ad_id": "campaign_12345",
      "timestamp": "2026-02-20",
      "action": "click"
    }
  ]
}
```

### Step 7: Off-Facebook Activity Management

Facebook tracks activity from partner websites and apps to deliver targeted ads. The "Off-Facebook Activity" tool provides transparency and control:

1. **Clear History** — Disconnects past off-Facebook activity from your account
2. **Future Off-Facebook Activity** — Manage whether Facebook receives data from future browser/app activity

For privacy-conscious users, disabling "Future Off-Facebook Activity" prevents Facebook from receiving data about your browsing, though this may reduce ad relevance and some platform features.

### Step 8: API Rate Limits and Developer Controls

For developers building Facebook-integrated applications, understanding rate limits is critical:

| Endpoint Category | Daily Limit | Hourly Limit |
|-------------------|-------------|--------------|
| User Data (READ) | 600 calls | 200 calls |
| Publish (WRITE) | 200 calls | 50 calls |
| Search | 100 calls | 25 calls |

Implement exponential backoff in your applications to handle rate limit errors gracefully:

```python
import time
import requests

def facebook_api_call(url, token, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params={'access_token': token})

        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limited
            wait_time = (2 ** attempt) * 60
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            response.raise_for_status()

    raise Exception("Max retries exceeded")
```

### Step 9: Two-Factor Authentication

Secure your account with two-factor authentication (2FA) under **Settings → Security and Login → Two-factor authentication**. Facebook supports:

- **Authentication app** (recommended) — Use apps like Google Authenticator or Authy
- **Text message (SMS)** — Less secure, susceptible to SIM swapping
- **Recovery codes** — Generate backup codes for account recovery

For developers, enabling 2FA on developer accounts is mandatory for accessing certain API features and advertising tools.

### Step 10: Privacy Checkup Tool

Facebook's Privacy Checkup walks through key settings in a guided flow. Access it from your profile's "..." menu or navigate directly to the Privacy Checkup section. This tool is particularly useful for reviewing post audience defaults, checking profile visibility, adjusting tagging controls, and reviewing app permissions.

## Advanced: AccountKit and Login API

For developers implementing Facebook Login, the 2026 API introduces stricter consent requirements:

- Explicit opt-in for each data type accessed
- Mandatory explanation of data use
- User-facing data use dialog requirements

```javascript
// Facebook Login with granular permissions (2026)
FB.login(function(response) {
  if (response.authResponse) {
    // Request minimal permissions first
    FB.api('/me/permissions', 'DELETE', { permission: 'user_posts' });
  }
}, {
  scope: 'public_profile,email',
  return_scopes: true,
  extras: {
    setup: {
      guest_inputs: ['email']
    }
  }
});
```
---

### Step 11: Privacy Controls by Facebook Feature (2026 Edition)

Facebook has fragmented into multiple services, each with distinct privacy concerns. Here's the complete breakdown:

### Facebook Feed and Post Visibility

| Setting | Location | Impact | Recommendation |
|---------|----------|--------|-----------------|
| **Audience of new posts** | Settings → Privacy → Who can see your posts | Controls default visibility | Set to "Friends" |
| **Limit past posts** | Settings → Privacy → Limit who can see past posts | Changes all old public posts to Friends-only | Click once on any account |
| **Who can follow you** | Settings → Privacy → Who can follow you | Public follow (like Twitter) vs. friends-only | Set to "Friends" if private |
| **Who can see your friend list** | Settings → Privacy → Who can see your friend list | Visibility of connections | Set to "Friends only" |

**Real impact**: A single public post from 2015 could still be visible to strangers in 2026. Running "Limit past posts" takes 30 seconds and blocks all pre-2024 public posts.

### Messenger Privacy

Messenger has its own privacy settings separate from your main profile:

```javascript
// Messenger privacy configuration checklist
const messengerSettings = {
  "message_requests": {
    "setting": "Settings > Privacy > Message Requests",
    "options": [
      "Everyone (receives from strangers)",
      "Friends (messages only from friends)",
      "Friends except... (custom exclusion)"
    ],
    "recommendation": "Friends",
    "reason": "Prevents unknown users from initiating contact"
  },
  "read_receipts": {
    "setting": "Settings > Privacy > Read Receipts",
    "options": ["On", "Off"],
    "recommendation": "Off",
    "reason": "Others cannot see when you read their messages"
  },
  "typing_indicators": {
    "setting": "Settings > Privacy > Typing Indicators",
    "options": ["On", "Off"],
    "recommendation": "Off",
    "reason": "Others cannot see that you are typing a response"
  },
  "activity_status": {
    "setting": "Settings > Privacy > Active Status",
    "options": ["Show when online", "Hide from all", "Custom list"],
    "recommendation": "Hide from all",
    "reason": "Facebook won't broadcast when you're active online"
  }
};
```

**Critical finding**: Messenger collects your location data if you allow it. Location is separate from your main Facebook privacy settings.

### Instagram Integration (if linked)

Facebook owns Instagram. If your accounts are linked:
- Facebook can access your Instagram data
- Instagram audience settings don't override Facebook privacy defaults
- Unlink accounts to prevent data sharing:

```javascript
// Unlinking Instagram from Facebook
// Facebook > Settings > Apps and Websites > Instagram
// Click Instagram > Remove
// Confirmation: "Your Instagram and Facebook accounts are no longer connected"

// After unlinking:
// - Facebook cannot access your Instagram data
// - Your Instagram audience is controlled by Instagram settings only
// - Note: Facebook still owns Instagram, but can't cross-pollinate your data
```

---

### Step 12: Data Request and Export for Privacy Audit

Understand exactly what Facebook knows about you:

```bash
#!/bin/bash
# facebook_data_audit.sh

echo "=== Facebook Data Audit Script ==="

# Step 1: Access Facebook data download
echo "1. Go to Settings > Your Facebook Information > Download Your Information"

# Step 2: Select data categories to export
echo "2. Select these minimum categories:"
echo "   - Profile Information (name, birthdate, bio)"
echo "   - Posts (all your posts ever)"
echo "   - Photos (profile, timeline, tagged)"
echo "   - Comments (all comments you made)"
echo "   - Messages (Messenger history)"
echo "   - Ads (all ads you interacted with)"
echo "   - Cookies (tracking data)"

# Step 3: Choose format and initiate download
echo "3. Select JSON format (machine-readable)"
echo "4. Click 'Create Download' (may take 24-48 hours)"

# Step 4: Download the ZIP file when ready
# Step 5: Extract and analyze

# Parse the JSON to find concerning data points
python3 << 'PYTHON'
import json
import os

# Load the downloaded JSON files
with open("your-facebook-data/ads/ads_and_businesses/ads_interests.json") as f:
    ads_data = json.load(f)

# Analyze what categories Facebook thinks you are
print("=== Facebook's Interest Categories for Targeting ===")
if "interests" in ads_data:
    interests = ads_data["interests"]
    # Sort by category count
    categories = {}
    for interest in interests:
        category = interest.get("category", "unknown")
        categories[category] = categories.get(category, 0) + 1

    for category, count in sorted(categories.items(), key=lambda x: x[1], reverse=True):
        print(f"{category}: {count} interests")

print("\n=== Sample Interests ===")
for interest in interests[:10]:
    print(f"  - {interest.get('name', 'Unknown')}")
PYTHON

echo "4. Review findings:"
echo "   - Are there interests you don't recognize?"
echo "   - Does Facebook's profile match your real interests?"
echo "   - Are there sensitive categories you don't want targeted?"

# Step 6: Remove misclassified interests
echo "5. Go to Settings > Ads > Ad Preferences > Your Interests"
echo "   Remove any interests you don't want to be targeted for"
```

---

### Step 13: Developer-Specific Privacy Hardening

If you build applications that use Facebook Login or integrate with Facebook data:

### Minimal Permissions Request

```javascript
// WRONG: Request too many permissions
FB.login(function(response) {}, {
  scope: 'public_profile,email,user_friends,user_photos,user_posts,user_location,user_birthday'
});

// CORRECT: Request only what you need
FB.login(function(response) {}, {
  scope: 'public_profile,email',
  return_scopes: true,
  auth_type: 'rerequest'  // Forces re-consent dialog if user previously denied
});

// BEST: Request permissions on-demand as features are needed
function enablePhotoFeature() {
  FB.login(function(response) {
    if (response.authResponse) {
      // User consented to photo access
      fetchUserPhotos();
    }
  }, {
    scope: 'user_photos'
  });
}
```

### Revoking Unused Permissions

Users often grant permissions they later regret:

```bash
# Revoke specific permissions without logging user out
curl -X DELETE \
  "https://graph.facebook.com/USER_ID/permissions?permission=user_photos&access_token=YOUR_ACCESS_TOKEN"

# After revocation: user_photos permission is permanently revoked
# Next API call attempting to read photos will fail with permission error
```

### Monitoring Permission Changes

If you maintain a user database, monitor when permissions change:

```python
import json
from datetime import datetime

def track_permission_changes(user_id, old_permissions, new_permissions):
    """Log when a user revokes or grants permissions"""

    lost_permissions = set(old_permissions) - set(new_permissions)
    new_permissions_granted = set(new_permissions) - set(old_permissions)

    if lost_permissions:
        log_event({
            "timestamp": datetime.utcnow().isoformat(),
            "event": "permission_revoked",
            "user_id": user_id,
            "permissions": list(lost_permissions)
        })

        # Example: If user revoked photos permission, delete cached photos
        if 'user_photos' in lost_permissions:
            delete_cached_user_photos(user_id)

    if new_permissions_granted:
        log_event({
            "timestamp": datetime.utcnow().isoformat(),
            "event": "permission_granted",
            "user_id": user_id,
            "permissions": list(new_permissions_granted)
        })
```

---

## Privacy Comparison: Facebook vs. Alternatives (2026)

| Platform | Data Collection | Encryption | Business Model | Privacy Rating |
|----------|-----------------|-----------|-----------------|-----------------|
| **Facebook** | Extensive (profile, behavior, off-Facebook activity) | HTTPS only | Advertising | 2/5 |
| **Signal** | Minimal (phone number, encrypted messages) | End-to-end encryption | Non-profit | 5/5 |
| **Telegram** | Moderate (chats not encrypted by default) | Optional E2E | Freemium | 3/5 |
| **WhatsApp** | Minimal but owned by Meta | End-to-end encryption | Freemium (Meta) | 3/5 |
| **Mastodon** | Self-hosted privacy control | HTTPS only | Open source | 4/5 |

**Reality check**: Moving away from Facebook entirely is the most effective privacy measure. However, if you must maintain a Facebook account for social/professional reasons, aggressive privacy settings reduce (but don't eliminate) exposure.

---

### Step 14: Annual Facebook Privacy Audit Checklist

Run this checklist once per year:

```markdown
### Step 15: Facebook Privacy Audit (Annual)

- [ ] Settings > Privacy > Review post audience default (should be Friends)
- [ ] Settings > Privacy > Run "Limit past posts" (blocks all old public posts)
- [ ] Settings > Apps and Websites > Review connected apps, revoke unused ones
- [ ] Settings > Privacy > Check "Who can see your friend list" (Friends only)
- [ ] Settings > Privacy > Check "Who can follow you" (Friends or disabled)
- [ ] Messenger > Settings > Privacy > Check message request filtering
- [ ] Ads > Ad Preferences > Remove unwanted interest categories
- [ ] Settings > Your Facebook Information > Download data and review
- [ ] Settings > Security > Review login activity (check for unusual locations)
- [ ] Settings > Security > Enable Two-Factor Authentication if not already enabled
- [ ] Posts > Check any public posts older than 2 years, delete or make private

**Time investment**: 15 minutes
**Privacy improvement**: Significant (60-70% reduction in data exposure)
```

---

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to complete this setup?**

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

- [Facebook Marketplace Privacy Settings Guide](/privacy-tools-guide/facebook-marketplace-privacy-settings-guide/)
- [Harden Macos Sequoia Privacy Settings Beyond Default](/privacy-tools-guide/how-to-harden-macos-sequoia-privacy-settings-beyond-default-configuration-complete-guide/)
- [iOS Journal App Privacy Settings Explained: A Complete Guide](/privacy-tools-guide/ios-journal-app-privacy-settings-explained/)
- [IOS Privacy Settings: Complete Walkthrough](/privacy-tools-guide/ios-privacy-settings-complete-walkthrough-every-toggle-expla/)
- [Ios Privacy Settings Complete Walkthrough Every Toggle.](/privacy-tools-guide/ios-privacy-settings-complete-walkthrough-every-toggle-explained/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
