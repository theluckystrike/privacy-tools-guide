---
layout: default
title: "LinkedIn Deceased Member Profile Removal How To Report"
description: "Report a deceased LinkedIn member's profile through the memorial request form in profile settings or via LinkedIn's support page. Provide a death certif..."
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Privacy Tools Guide"
permalink: /linkedin-deceased-member-profile-removal-how-to-report-and-m/
categories: [guides]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Report a deceased LinkedIn member's profile through the memorial request form in profile settings or via LinkedIn's support page. Provide a death certificate, obituary, or news article as proof. LinkedIn can memorialize (make read-only, add "Remembering" label) or delete the account if requested. For your own profile, designate a legacy contact in Account Settings > Legacy Settings to give someone permission to memorialize or manage your profile after death.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1 - Understand LinkedIn's Deceased Member Policy

LinkedIn provides two options for handling deceased members' accounts: memorialization or complete removal. Memorialization transforms the profile into a tribute page where connections can leave condolences and celebrate the person's professional legacy. Complete removal deletes the profile and associated data entirely.

The distinction matters for different use cases. Families often prefer memorialization to preserve professional achievements and network connections. Organizations handling estate matters might prefer complete removal, especially when the deceased had sensitive professional information or proprietary code in their profile.

LinkedIn's deceased member form serves as the official entry point for both options. The process requires documentation and varies depending on your relationship to the deceased.

Step 2 - How to Report a Deceased Member's Profile

The primary method uses LinkedIn's dedicated Deceased Member Form. You'll need to provide:

- The LinkedIn profile URL of the deceased member
- Your relationship to the deceased (family member, connection, or authorized representative)
- Proof of death (obituary link, death certificate, or news article)
- Your contact information for follow-up

For family members requesting memorialization, LinkedIn typically requires one of the following:
- Link to an obituary or death notice
- Copy of the death certificate
- News article announcing the death
- Social Security Death Index confirmation

Here's a bash script that validates a LinkedIn profile URL format before submission:

```bash
#!/bin/bash
Validate LinkedIn profile URL format

validate_linkedin_url() {
    local url="$1"

    # Check for valid LinkedIn profile URL pattern
    if [[ $url =~ ^https?://(www\.)?linkedin\.com/in/[a-zA-Z0-9-]+/?$ ]]; then
        echo "Valid LinkedIn profile URL"
        return 0
    else
        echo "Invalid LinkedIn profile URL format"
        echo "Expected format: https://www.linkedin.com/in/username/"
        return 1
    fi
}

Usage example
validate_linkedin_url "https://www.linkedin.com/in/john-doe-123456789/"
```

Step 3 - The Memorialization Process

After submitting the deceased member form, LinkedIn's Trust and Safety team reviews the request. Processing times vary based on volume, but typically take 5-15 business days. During this period, the original profile remains visible.

When memorialized, the profile undergoes several changes:
- The word "Memorialized" appears on the profile
- The profile becomes non-searchable
- Login credentials are deactivated
- Private messaging is disabled
- Comments and reactions remain visible as tributes

For developers building tools that interact with LinkedIn profiles, memorialized profiles return different API responses. If you're maintaining a system that scrapes or monitors LinkedIn profiles, account for this state:

```python
import requests

def check_profile_status(profile_url, session):
    """Check if a LinkedIn profile is memorialized."""

    # Note: LinkedIn API restrictions apply
    # This is a conceptual example for educational purposes

    headers = {
        "Authorization": f"Bearer {session.access_token}",
        "X-Restli-Protocol-Version": "2.0.0"
    }

    # Extract profile ID from URL
    profile_id = profile_url.split("/in/")[-1].strip("/")

    response = session.get(
        f"https://api.linkedin.com/v2/people/{profile_id}",
        headers=headers
    )

    if response.status_code == 200:
        data = response.json()
        # Check for memorialization status
        is_memorialized = data.get("headline", "").endswith("(Memorialized)")
        return {
            "exists": True,
            "memorialized": is_memorialized,
            "data": data
        }

    return {"exists": False, "error": response.text}
```

Step 4 - Bulk Requests for Organizations

HR departments and organizations managing employee departures sometimes need to handle multiple profiles. LinkedIn's policy requires individual requests for each profile, but you can improve the documentation process:

```python
Generate documentation for bulk deceased member requests

def prepare_submission_data(employees_file):
    """Prepare batch submission data for multiple profiles."""

    import csv

    submissions = []

    with open(employees_file, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            submissions.append({
                "profile_url": row['linkedin_url'],
                "full_name": row['full_name'],
                "death_date": row['death_date'],
                "documentation_link": row['obituary_url'],
                "relationship": row['relationship'],  # family, colleague
                "contact_email": row['contact_email']
            })

    return submissions

Example usage
data = prepare_submission_data("deceased_employees.csv")
for item in data:
    print(f"Profile: {item['profile_url']}")
    print(f"Documentation: {item['documentation_link']}")
```

Maintain thorough records of all submissions, including confirmation numbers and correspondence with LinkedIn's team. This documentation matters for estate administration and legal compliance.

Step 5 - Alternative Request Methods

If the standard form isn't accessible or you've encountered issues, LinkedIn provides secondary contact methods:

1. LinkedIn Help Community - Post in the LinkedIn Help Community under the "Managing an Account" category. Community moderators can escalate memorialization requests.

2. Legal Requests - For cases requiring immediate action or involving legal disputes, LinkedIn accepts formal legal requests through their Legal Claims page.

3. Premium Support - If you have a LinkedIn Premium subscription associated with the deceased's account, premium support may provide faster response times.

Step 6 - Privacy Considerations for Developers

When building systems that interact with LinkedIn data, consider these privacy-aware practices:

- Never store profile data of deceased individuals without explicit authorization
- Implement automated checks to flag memorialized profiles in your datasets
- Respect LinkedIn's terms of service regarding deceased member data
- Provide clear data retention policies that address deceased user data

If you're building recruitment tools or professional networking analyzers, add functionality to handle memorialized profiles gracefully:

```javascript
// JavaScript example for handling memorialized profiles

function processLinkedInProfile(profile) {
    const isMemorialized = profile.headline?.includes('Memorialized') ||
                          profile.firstName?.includes('(Deceased)');

    if (isMemorialized) {
        return {
            status: 'memorialized',
            shouldExclude: true,
            reason: 'Profile belongs to deceased individual',
            metadata: {
                memorializedDate: profile.lastModified,
                originalName: profile.firstName.replace('(Deceased)', '').trim()
            }
        };
    }

    return { status: 'active', shouldExclude: false };
}
```

Step 7 - What LinkedIn Does Not Allow

Understanding limitations prevents wasted effort:

- You cannot access the deceased's private messages or connection list
- Connection transfers are not possible
- Profile editing requires account access, which requires login credentials
- LinkedIn does not provide bulk memorialization APIs

For families seeking to preserve professional portfolios, consider archiving the profile manually through screenshots or web archiving tools before requesting memorialization.

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to report?

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

- [How To Protect LinkedIn Profile From Being Discovered](/how-to-protect-linkedin-profile-from-being-discovered-by-dat/)
- [How To Opt Out Of LinkedIn Data Being Used For Ai Training](/how-to-opt-out-of-linkedin-data-being-used-for-ai-training-a/)
- [Android Work Profile Privacy Separation Guide](/android-work-profile-privacy-separation-guide/)
- [How To Verify Dating Profile Authenticity Without Revealing](/how-to-verify-dating-profile-authenticity-without-revealing-/)
- [Using curl for LinkedIn API](/social-media-data-request-download-guide-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
