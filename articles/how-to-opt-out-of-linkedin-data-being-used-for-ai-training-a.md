---
layout: default
title: "How To Opt Out Of LinkedIn Data Being Used For Ai Training"
description: "A practical guide for developers and power users on how to prevent LinkedIn from using your professional data for AI training and machine learning"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-opt-out-of-linkedin-data-being-used-for-ai-training-a/
categories: [guides, security]
tags: [privacy-tools-guide, artificial-intelligence]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

LinkedIn has increasingly turned to AI training as a way to improve its platform and potentially monetize user data. If you're a developer or power user concerned about your professional information being used to train machine learning models, there are specific steps you can take to opt out. This guide walks you through the available methods, their limitations, and additional privacy measures you can implement.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1 - Understand LinkedIn's AI Data Usage

LinkedIn's privacy policy outlines several ways the platform uses user data, including for "research and development" purposes. While the company has introduced some opt-out mechanisms, the implementation has been inconsistent, and many users remain unaware that their data may be contributing to AI model training.

The types of data potentially used include:
- Profile information (name, job title, company, skills)
- Connection networks
- Posts and articles you publish
- Messages and InMail content
- Profile views and search activity
- Endorsements and recommendations

Step 2 - The Official Opt-Out Method

LinkedIn provides a privacy setting to control data usage for AI training. Here's how to access it:

1. Click your profile photo and select Settings & Privacy
2. Navigate to the Privacy tab
3. Scroll to the Data for AI training and model development section
4. Toggle off the option to allow LinkedIn to use your data for AI training

This setting controls whether your content and data can be used to train LinkedIn's AI models. However, there are important caveats:

- The setting may only apply to LinkedIn's own AI features, not necessarily to data shared with third parties
- Historical data collected before you enabled the opt-out may still be retained
- The opt-out doesn't affect data already processed or used in models

Step 3 - Implement Programmatic Verification of Your Settings

For developers who want to verify their privacy settings programmatically, LinkedIn provides limited API access. You can check your account settings through the LinkedIn API:

```python
import requests

def check_ai_training_optout(access_token):
    """Check if user has opted out of AI training data usage"""
    headers = {
        'Authorization': f'Bearer {access_token}',
        'X-Restli-Protocol-Version': '2.0.0'
    }

    response = requests.get(
        'https://api.linkedin.com/v2/me',
        headers=headers
    )

    if response.status_code == 200:
        profile = response.json()
        # Check for privacy-related flags
        ai_training_consent = profile.get('aiTrainingOptIn', True)
        return {
            'opted_out': not ai_training_consent,
            'raw_setting': ai_training_consent
        }
    return None
```

Note that LinkedIn's API access is restricted and requires approval for most use cases. This example demonstrates the concept rather than providing working code.

Step 4 - Data Portability and Deletion Requests

Beyond the AI training opt-out, you can request a copy of your data or deletion under various privacy regulations. LinkedIn provides these options through their settings:

To request your data:
1. Go to Settings & Privacy > Privacy > How LinkedIn uses your data
2. Click Get a copy of your data

To submit a deletion request:
1. Navigate to Settings & Privacy > Privacy > How LinkedIn uses your data
2. Click Request deletion of your account

For EU users, GDPR provides additional rights. You can submit a formal request through LinkedIn's GDPR portal, which typically requires a 30-day response timeline.

Step 5 - Automate Data Export with Selenium

For power users who want data backups, you can automate the export process using Selenium:

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def download_linkedin_data(email, password):
    """Automated data download request"""
    driver = webdriver.Chrome()
    driver.get('https://www.linkedin.com/login')

    # Login process
    driver.find_element(By.ID, 'username').send_keys(email)
    driver.find_element(By.ID, 'password').send_keys(password)
    driver.find_element(By.ID, 'password').submit()

    # Navigate to data download
    WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.LINK_TEXT, 'Settings & Privacy'))
    )

    driver.get('https://www.linkedin.com/psettings/privacy/data')

    # Click request download
    download_button = WebDriverWait(driver, 10).until(
        EC.element_to_be_clickable((By.XPATH,
            '//button[contains(text(), "Get a copy of your data")]'))
    )
    download_button.click()

    driver.quit()
```

This script requires the Selenium WebDriver and should be used responsibly, respecting LinkedIn's terms of service.

Step 6 - Hardening Your LinkedIn Profile

While you cannot completely prevent LinkedIn from collecting data, you can minimize what appears on your profile:

Profile visibility settings:
- Go to Settings & Privacy > Visibility
- Limit profile viewing options to restrict who sees your information
- Disable "Open to Work" visibility if you're not actively job searching

Content restrictions:
- Avoid posting detailed technical articles that might be scraped
- Keep work history generalized rather than detailed
- Remove unnecessary personal information from your profile

Step 7 - Monitor Third-Party App Permissions

LinkedIn integrates with numerous third-party applications that may have their own data practices:

1. Navigate to Settings & Privacy > Partners and services
2. Review all connected applications
3. Remove access for any apps you no longer use
4. Limit permissions for remaining apps to minimum necessary access

Many third-party LinkedIn apps have been discovered collecting data beyond their stated purposes, so regular audits are essential.

Step 8 - API-Based Privacy Automation

For developers building privacy-focused applications, you can monitor LinkedIn privacy settings using browser automation tools. This example uses Playwright to check privacy settings:

```javascript
const { chromium } = require('playwright');

async function checkPrivacySettings(credentials) {
    const browser = await chromium.launch();
    const page = await browser.newPage();

    await page.goto('https://www.linkedin.com/login');
    await page.fill('#username', credentials.email);
    await page.fill('#password', credentials.password);
    await page.click('#password');

    await page.waitForNavigation();
    await page.goto(
        'https://www.linkedin.com/psettings/privacy/ai-training'
    );

    // Check if opt-out toggle exists and its state
    const aiTrainingToggle = await page.$(
        'input[type="checkbox"][data-test-privacy-consent]'
    );

    if (aiTrainingToggle) {
        const isChecked = await aiTrainingToggle.isChecked();
        console.log(`AI Training opt-out status: ${!isChecked ? 'OPTED OUT' : 'OPTED IN'}`);
    }

    await browser.close();
}
```

Step 9 - Legal Options and Regulatory recourse

If you're located in certain jurisdictions, you have additional legal options:

- EU/EEA: File a GDPR complaint with your local data protection authority
- California: Invoke CCPA rights to know, delete, and opt-out of sale
- General: Request data deletion under applicable privacy laws

LinkedIn's privacy policy specifies a designated privacy officer for such requests. Document all communications and keep records of your opt-out attempts.

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to opt out of linkedin data being used for ai training?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Opt Out of Data Sharing Under Connecticut Data Privacy Act](/how-to-opt-out-of-data-sharing-under-connecticut-data-privac/)
- [Using curl for LinkedIn API](/social-media-data-request-download-guide-2026/)
- [How To Remove Personal Information From Ai Training Datasets](/how-to-remove-personal-information-from-ai-training-datasets/)
- [How To Protect LinkedIn Profile From Being Discovered](/how-to-protect-linkedin-profile-from-being-discovered-by-dat/)
- [How to Remove Personal Data from Data Brokers 2026:](/how-to-remove-personal-data-from-data-brokers/---)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
