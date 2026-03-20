---

layout: default
title: "How To Make Facebook Profile Private 2026"
description: "Learn how to make your Facebook profile private in 2026 using the Facebook Graph API, browser automation, and manual settings. Complete guide for."
date: 2026-03-15
author: theluckystrike
permalink: /how-to-make-facebook-profile-private-2026/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

If you're a developer or power user looking to lock down your Facebook presence, the platform's privacy controls have evolved significantly. While Facebook still offers manual settings, programmatic access through the Graph API and browser automation provide powerful options for bulk management and automation. This guide covers both approaches to make your Facebook profile private in 2026.

## Understanding Facebook's Privacy Architecture

Facebook organizes privacy settings across several categories: profile visibility, audience selection, tagging controls, and connection visibility. The platform uses a tiered audience system: Public, Friends, Friends Except Acquaintances, Specific Friends, and Only Me. Understanding these layers is essential for developers building tools that interact with Facebook's privacy controls.

The Facebook Graph API provides the most reliable programmatic approach for managing privacy settings. However, Facebook has restricted many privacy-related APIs, so some automation requires browser-based solutions using tools like Playwright or Selenium.

## Method 1: Using the Facebook Graph API

The Graph API remains the primary method for programmatic Facebook interactions. To access privacy settings, you'll need a Facebook Developer account and an access token with appropriate permissions.

### Step 1: Obtain an Access Token

For personal profile management, you can use the Facebook Login flow to obtain a user access token. The minimal required permissions for privacy reading are:

```
permissions[]=pages_read_engagement
permissions[]=pages_manage_posts
```

For full privacy management, you'll need additional permissions that Facebook now restricts. Many privacy settings require manual verification or are not available via API for personal accounts.

### Step 2: Query Current Privacy Settings

Once you have a valid token, query your profile's privacy settings:

```bash
curl -i -X GET "https://graph.facebook.com/v21.0/me?fields=id,name,email&access_token=YOUR_ACCESS_TOKEN"
```

For page-level settings, you can retrieve privacy information:

```bash
curl -i -X GET "https://graph.facebook.com/v21.0/PAGE_ID?fields=privacy&access_token=YOUR_ACCESS_TOKEN"
```

The response includes privacy settings in JSON format:

```json
{
  "privacy": {
    "value": "EVERYONE",
    "description": "Who can see your profile",
    "allow": "0",
    "deny": "0"
  }
}
```

### Step 3: Update Privacy Settings via API

For pages and posts, you can update privacy programmatically:

```bash
curl -i -X POST "https://graph.facebook.com/v21.0/PAGE_ID/privacy" \
  -d "value=EVERYONE" \
  -d "access_token=YOUR_ACCESS_TOKEN"
```

Note: For personal profiles, Facebook restricts direct API modification of most privacy settings. This is where browser automation becomes necessary.

## Method 2: Browser Automation with Playwright

For managing personal profile settings that the API restricts, browser automation provides a viable alternative. The following example uses Playwright to navigate Facebook's settings interface.

### Setup and Dependencies

```javascript
const { chromium } = require('playwright');

async function setFacebookPrivacy() {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();
  
  // Navigate to Facebook login
  await page.goto('https://www.facebook.com/login');
  
  // Perform login (credentials should come from environment variables)
  await page.fill('#email', process.env.FB_EMAIL);
  await page.fill('#pass', process.env.FB_PASSWORD);
  await page.click('#loginbutton');
  
  // Wait for navigation to complete
  await page.waitForNavigation();
  
  // Navigate to privacy settings
  await page.goto('https://www.facebook.com/settings?tab=privacy');
  
  // Adjust audience for future posts
  await page.click('text=Who can see your future posts?');
  await page.click('text=Friends'); // Select friends-only
  
  // Limit past posts
  await page.click('text=Limit who can see past posts');
  await page.click('text=Limit Past Posts');
  
  await browser.close();
}

setFacebookPrivacy().catch(console.error);
```

This script demonstrates the core pattern: authenticate, navigate to settings, and interact with the appropriate UI elements. Facebook's UI changes frequently, so you'll need to inspect the current DOM structure and adjust selectors accordingly.

## Method 3: Manual Settings for Maximum Privacy

For users who prefer manual configuration, Facebook's privacy settings are accessible through the Settings & Privacy menu. Here's the configuration checklist for maximum privacy:

### Profile Visibility Settings

Navigate to Settings > Privacy > Your Profile. Set each option according to your preferences:

- **Who can see your friend list**: Only Me or Friends
- **Who can see the people, Pages and lists you follow**: Only Me
- **Who can see your birth date**: Only Me (for both month/day and year)
- **Who can see your contact information**: Only Me for all fields

### Timeline and Tagging

- **Who can post on your timeline**: Only Me
- **Who can see what others post on your timeline**: Friends
- **Allow tagging by friends**: Disabled or Friends only
- **Review posts you're tagged in before the post appears on your timeline**: Enabled

### Connection Visibility

- **Who can send you friend requests**: Friends of Friends
- **Who can see your friend requests**: No One
- **Who can look you up using the email address you provided**: Only Me
- **Who can look you up using the phone number you provided**: Only Me
- **Allow search engines outside of Facebook to link to your profile**: Disabled

## Security Hardening Beyond Privacy

Privacy and security go hand in hand. Beyond visibility settings, enable these security features:

### Two-Factor Authentication

```bash
# Check if 2FA is enabled via Graph API
curl -i -X GET "https://graph.facebook.com/v21.0/me?fields=two_factor_status&access_token=YOUR_ACCESS_TOKEN"
```

### Active Sessions Review

Regularly review and terminate unused sessions:

```javascript
// Example: List active sessions via Playwright
await page.goto('https://www.facebook.com/settings?tab=security');
const sessions = await page.$$eval('.uiListHelperClearfix li', 
  items => items.map(item => item.textContent));
console.log('Active sessions:', sessions);
```

### Login Alerts

Enable login alerts to receive notifications when your account is accessed from new devices. This appears in Settings > Security and Login > Get alerts about unrecognized logins.

## Automating Privacy Audits

For developers managing multiple accounts or conducting privacy audits, create a scheduled script that:

1. Logs into each account programmatically
2. Navigates to the privacy settings page
3. Captures screenshots of current configuration
4. Exports settings to a JSON report

```javascript
// privacy-audit.js
const fs = require('fs');

async function auditPrivacySettings(page) {
  await page.goto('https://www.facebook.com/settings?tab=privacy');
  
  // Extract all privacy settings
  const settings = await page.evaluate(() => {
    const sections = document.querySelectorAll('[data-testid="settings_section"]');
    const result = {};
    
    sections.forEach(section => {
      const label = section.querySelector('span')?.textContent;
      const value = section.querySelector('[role="button"]')?.textContent;
      if (label && value) {
        result[label] = value;
      }
    });
    
    return result;
  });
  
  return settings;
}

// Run audit and save to file
(async () => {
  // ... browser setup ...
  const auditResults = await auditPrivacySettings(page);
  fs.writeFileSync('privacy-audit.json', JSON.stringify(auditResults, null, 2));
  // ... cleanup ...
})();
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [Best Private Alternative to Google Drive 2026: A.](/privacy-tools-guide/best-private-alternative-to-google-drive-2026/)
- [Dashlane vs 1Password Comparison 2026: A Developer.](/privacy-tools-guide/dashlane-vs-1password-comparison-2026/)
- [How to Make Instagram Story Viewers List Private.](/privacy-tools-guide/how-to-make-instagram-story-viewers-list-private-controlling/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
