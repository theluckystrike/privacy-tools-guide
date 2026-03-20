---

layout: default
title: "Using curl for LinkedIn API"
description: "Learn how to request and download your social media data from major platforms. Includes API methods, automation scripts, and best practices for developers."
date: 2026-03-15
author: theluckystrike
permalink: /social-media-data-request-download-guide-2026/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Privacy regulations like GDPR and CCPA have given users powerful rights over their personal data. As a developer or power user, understanding how to exercise these rights—and automate the process—provides significant value. This guide covers data request mechanisms across major social platforms, with practical code examples for automating your data downloads.

## Understanding Data Requests

Most social media platforms provide two types of data access: self-service downloads through account settings and formal data requests under privacy regulations. Self-service downloads typically include your posts, media, and basic account information. Formal requests under GDPR Article 15 or CCPA can yield more data, including algorithmic predictions, behavioral profiles, and inferred attributes.

The distinction matters for developers. Self-service downloads use web interfaces or APIs you can access with standard authentication. Formal requests go through legal or privacy teams and may take 30-45 days to fulfill.

## Platform-Specific Data Request Methods

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

## Automating Cross-Platform Data Collection

For developers managing data across multiple platforms, a unified approach helps:

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

## Legal Considerations

When collecting data through formal privacy requests, platforms typically provide ZIP archives containing all data they hold. These archives can include:
- Account activity logs
- Message history
- Search history
- Ad interactions
- Inferred interests and demographics

Review platform-specific terms of service. Automated data collection should comply with API terms and rate limiting policies. For personal data under GDPR, you have the right to request a complete data portrait—use this right strategically.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [People Search Sites Opt Out Complete Guide 2026](/privacy-tools-guide/people-search-sites-opt-out-complete-guide-2026/)
- [Social Media Privacy Policy Comparison 2026: A Developer.](/privacy-tools-guide/social-media-privacy-policy-comparison-2026/)
- [How to Demand a Company Stop Selling Your Personal Data.](/privacy-tools-guide/how-to-demand-company-stop-selling-your-personal-data-under-/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
