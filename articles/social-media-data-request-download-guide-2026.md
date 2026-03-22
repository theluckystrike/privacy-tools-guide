---
layout: default
title: "Using curl for LinkedIn API"
description: "Learn how to request and download your social media data from major platforms. Includes API methods, automation scripts, and best practices for developers"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /social-media-data-request-download-guide-2026/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Privacy regulations like GDPR and CCPA have given users powerful rights over their personal data. As a developer or power user, understanding how to exercise these rights—and automate the process—provides significant value. This guide covers data request mechanisms across major social platforms, with practical code examples for automating your data downloads.

# Use .env files with dotenv
export FACEBOOK_TOKEN="your_token_here"
export TWITTER_BEARER="your_bearer_token"
{% endhighlight %}

2.
- **Most developers use OAuth**: with specific scope grants for advertising or personal data retrieval.
- **How do I get**: started quickly? Pick one tool from the options discussed and sign up for a free trial.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **Free tier access provides**: limited endpoints, while Academic Research or Enterprise access offers data exports.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand Data Requests

Most social media platforms provide two types of data access: self-service downloads through account settings and formal data requests under privacy regulations. Self-service downloads typically include your posts, media, and basic account information. Formal requests under GDPR Article 15 or CCPA can yield more data, including algorithmic predictions, behavioral profiles, and inferred attributes.

The distinction matters for developers. Self-service downloads use web interfaces or APIs you can access with standard authentication. Formal requests go through legal or privacy teams and may take 30-45 days to fulfill.

### Step 2: Platform-Specific Data Request Methods

### Facebook/Meta

Meta provides data downloads through the "Download Your Information" tool in Account Settings. For developers seeking automated access, the Graph API offers programmatic retrieval:

{% highlight javascript %}
const axios = require('axios');

async function downloadFacebookData(accessToken, dataType) {
 const response = await axios.get(
 `https://graph.facebook.com/v18.0/me/${dataType}`,
 {
 params: {
 access_token: accessToken,
 fields: 'id,message,created_time,permalink_url'
 },
 responseType: 'stream'
 }
 );
 return response.data;
}

// Usage: downloadFacebookData(token, 'posts')
{% endhighlight %}

For data exports, use the Data Download tool through their OAuth flow. Navigate to Settings > Your Facebook Information > Download Your Information to initiate a request programmatically using the Marketing API.

### Twitter/X

X (formerly Twitter) provides data downloads through the "Your X data" section in settings. Developers can use the API v2 for real-time data retrieval:

{% highlight python %}
import requests
from datetime import datetime, timedelta

def get_twitter_data(bearer_token, endpoint, params=None):
 headers = {
 "Authorization": f"Bearer {bearer_token}",
 "Content-Type": "application/json"
 }
 url = f"https://api.twitter.com/2/{endpoint}"
 response = requests.get(url, headers=headers, params=params)
 return response.json()

# Fetch recent tweets
params = {
 "tweet.fields": "created_at,public_metrics",
 "max_results": 100
}
tweets = get_twitter_data(bearer_token, "users/me/tweets", params)
{% endhighlight %}

Note that X's API access requires approval through their developer portal. Free tier access provides limited endpoints, while Academic Research or Enterprise access offers data exports.

### Instagram

Instagram shares the Meta ecosystem, so data requests route through Facebook's framework. For API-based access, you'll need an Instagram Basic Display API token:

{% highlight python %}
import requests

def get_instagram_media(access_token):
 url = f"https://graph.instagram.com/me/media"
 params = {
 "fields": "id,caption,media_type,media_url,timestamp,permalink",
 "access_token": access_token
 }
 response = requests.get(url, params=params)
 return response.json()

# Paginate through results
def get_all_media(access_token):
 all_media = []
 url = "https://graph.instagram.com/me/media"

 while url:
 params = {
 "fields": "id,caption,media_type,media_url,timestamp,permalink,next_page",
 "access_token": access_token
 }
 response = requests.get(url, params=params).json()
 all_media.extend(response.get("data", []))
 url = response.get("paging", {}).get("next")

 return all_media
{% endhighlight %}

### LinkedIn

LinkedIn's data export is available through the "Getting a copy of your data" option in Account Settings. For developers, the LinkedIn Marketing API provides advertising data retrieval:

{% highlight bash %}
# Using curl for LinkedIn API
curl -X GET "https://api.linkedin.com/v2/ugcPosts" \
 -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
 -H "X-Restli-Protocol-Version: 2.0.0"
{% endhighlight %}

Note that LinkedIn's API access is heavily restricted. Most developers use OAuth with specific scope grants for advertising or personal data retrieval.

### Reddit

Reddit provides data downloads through the "Download" tab in User Settings. The Reddit API enables programmatic access:

{% highlight python %}
import praw

reddit = praw.Reddit(
 client_id="YOUR_CLIENT_ID",
 client_secret="YOUR_CLIENT_SECRET",
 user_agent="data-download-script/1.0"
)

def get_reddit_data(username):
 user = reddit.redditor(username)
 submissions = []

 for post in user.submissions.new(limit=None):
 submissions.append({
 "id": post.id,
 "title": post.title,
 "selftext": post.selftext,
 "created_utc": post.created_utc,
 "subreddit": str(post.subreddit)
 })

 return submissions
{% endhighlight %}

### Step 3: Automate Cross-Platform Data Collection

For developers managing data across multiple platforms, an unified approach helps:

{% highlight python %}
import asyncio
import aiohttp
from dataclasses import dataclass
from typing import List, Dict, Optional

@dataclass
class SocialDataConfig:
 platform: str
 access_token: str
 data_types: List[str]

async def collect_platform_data(config: SocialDataConfig) -> Dict:
 """Unified data collection across platforms."""
 collectors = {
 "facebook": collect_facebook_data,
 "twitter": collect_twitter_data,
 "instagram": collect_instagram_data,
 "reddit": collect_reddit_data
 }

 collector = collectors.get(config.platform)
 if not collector:
 raise ValueError(f"Unsupported platform: {config.platform}")

 results = {}
 for data_type in config.data_types:
 results[data_type] = await collector(config.access_token, data_type)

 return results

async def main():
 configs = [
SocialDataConfig("facebook", fb_token, ["posts", "photos"]),
SocialDataConfig("twitter", tw_token, ["tweets"]),
SocialDataConfig("reddit", rd_token, ["submissions"])
 ]

 tasks = [collect_platform_data(config) for config in configs]
 all_data = await asyncio.gather(*tasks)

 return all_data

# Run with: asyncio.run(main())
{% endhighlight %}

## Best Practices for Data Downloads

1. **Token Security**: Store access tokens in environment variables or secure vaults. Never commit tokens to version control:

{% highlight bash %}
# Use .env files with dotenv
export FACEBOOK_TOKEN="your_token_here"
export TWITTER_BEARER="your_bearer_token"
{% endhighlight %}

2. **Rate Limiting**: Respect platform rate limits. Implement exponential backoff:

{% highlight python %}
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1):
 def decorator(func):
 @wraps(func)
 def wrapper(*args, **kwargs):
 for attempt in range(max_retries):
 try:
 return func(*args, **kwargs)
 except RateLimitError:
 delay = base_delay * (2 ** attempt)
 time.sleep(delay)
 raise Exception("Max retries exceeded")
 return wrapper
 return decorator
{% end %}

3. **Data Storage**: Store downloaded data locally with encryption. Use formats like JSON or CSV for interoperability:

{% highlight python %}
import json
import hashlib
from pathlib import Path

def save_data_secure(data, filename, encryption_key=None):
 filepath = Path(filename)

 with open(filepath, 'w') as f:
 json.dump(data, f, indent=2, default=str)

 # Verify integrity
 with open(filepath, 'rb') as f:
 checksum = hashlib.sha256(f.read()).hexdigest()

 return {"path": str(filepath), "checksum": checksum}
{% endhighlight %}

4. **Regular Sync**: Schedule periodic downloads to maintain current archives. Use cron jobs or CI/CD pipelines:

{% highlight bash %}
# Cron job for weekly data sync
0 2 * * 0 cd /path/to/project && python sync_social_data.py >> /var/log/data_sync.log 2>&1
{% endhighlight %}

### Step 4: Legal Considerations

When collecting data through formal privacy requests, platforms typically provide ZIP archives containing all data they hold. These archives can include:
- Account activity logs
- Message history
- Search history
- Ad interactions
- Inferred interests and demographics

Review platform-specific terms of service. Automated data collection should comply with API terms and rate limiting policies. For personal data under GDPR, you have the right to request a complete data portrait—use this right strategically.

### Download and Inspect Your Social Media Data

```bash
# Request your data archive from major platforms via their official APIs

# Twitter/X: request archive (triggers email with download link)
curl -X POST "https://api.twitter.com/2/users/:id/tweets"               -H "Authorization: Bearer $TWITTER_BEARER_TOKEN"

# LinkedIn: request data export (REST API, requires OAuth)
curl -s "https://api.linkedin.com/v2/me"               -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" | jq .

# Facebook: download your data via Graph API
curl -s "https://graph.facebook.com/me?fields=id,name,email"               -d "access_token=$FB_ACCESS_TOKEN" | jq .

# Parse downloaded JSON archive to count data categories
python3 -c "
import json, pathlib
for f in pathlib.Path('download').rglob('*.json'):
    data = json.loads(f.read_text())
    if isinstance(data, list):
        print(f'{f.name}: {len(data)} entries')
"
```

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Employee Social Media Privacy Can Employer Fire You For Priv](/privacy-tools-guide/employee-social-media-privacy-can-employer-fire-you-for-priv/)
- [How to Block Social Media Share Button Tracking on Websites](/privacy-tools-guide/how-to-block-social-media-share-button-tracking-on-websites/)
- [How To Create Anonymous Social Media Accounts](/privacy-tools-guide/how-to-create-anonymous-social-media-accounts/)
- [How To Delete Old Social Media Accounts](/privacy-tools-guide/how-to-delete-old-social-media-accounts/)
- [How To Prepare Social Media Accounts For Memorialization Com](/privacy-tools-guide/how-to-prepare-social-media-accounts-for-memorialization-com/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
