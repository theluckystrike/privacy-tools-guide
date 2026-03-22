---
layout: default
title: "How To Remove Yourself From True People Search Instant"
description: "A practical guide for developers and power users to remove personal data from people search sites. Includes automation scripts and API strategies"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-remove-yourself-from-true-people-search-instant-check/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Data broker websites like TruePeopleSearch, Instant Checkmate, and Radaris aggregate publicly available information and make it searchable to anyone with an internet connection. This includes your name, address, phone number, email, and in some cases, criminal records and background checks. For developers and power users concerned about privacy, removing your data from these platforms is a critical step in reclaiming your digital footprint.

This guide covers the manual opt-out processes for major people search sites and provides automation strategies to improve the removal of your data across multiple databases.

## Understanding How Data Brokers Collect Your Information

Before removing your data, understanding how these platforms acquire it helps you address the root source. Data brokers collect information from:

- **Public records**: Property deeds, voter registrations, court documents, and birth certificates
- **Social media**: Publicly accessible profiles and posts
- **Data sharing agreements**: Companies that sell user data to brokers
- **Online directories**: Business listings, professional networks, and subscription services

Each time you provide information to a service, there's a chance it flows into the data broker ecosystem. Removing yourself from these sites provides immediate relief, but new data can appear as brokers continuously update their databases.

## Removing Your Data from TruePeopleSearch

TruePeopleSearch offers a free people search service that displays personal information prominently. Their opt-out process requires submitting a manual request through their removal form.

### Manual Opt-Out Steps

1. Visit the TruePeopleSearch removal page at `truepeoplesearch.com/remove`
2. Enter your first name, last name, and state
3. Locate your listing in the search results
4. Click "Remove This Record" next to your information
5. Complete the verification process (email confirmation required)

The removal typically takes effect within 24-72 hours, though you may need to repeat this process if your information reappears.

## Removing Your Data from Instant Checkmate

Instant Checkmate operates as a background check service and maintains extensive records. Their opt-out process follows a formal request pattern.

### Manual Opt-Out Steps

1. Navigate to the Instant Checkmate opt-out page at `instantcheckmate.com/opt-out`
2. Enter your full name and date of birth
3. Search for your profile
4. Select your record and confirm removal
5. Verify your identity via email

Instant Checkmate states that removals are processed within 48 hours, but the site may retain records for legal and compliance purposes.

## Removing Your Data from Radaris

Radaris functions as both a people search engine and a data broker that supplies information to other services. Their opt-out requires account deletion.

### Manual Opt-Out Steps

1. Visit `radaris.com/page/remove`
2. Use the search function to locate your profile
3. Click "Delete" on your profile page
4. Confirm the deletion request via email

Radaris also offers a batch removal process for multiple records, which is useful if your information appears under several variations.

## Automating Data Removal with Scripts

For developers managing removal requests across multiple sites, Python scripts can automate parts of this process. The following example demonstrates a structured approach to tracking opt-out requests:

```python
import json
import datetime
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class RemovalRequest:
    site: str
    url: str
    status: str
    request_date: datetime.datetime
    completed_date: Optional[datetime.datetime] = None

    def to_dict(self):
        return {
            "site": self.site,
            "url": self.url,
            "status": self.status,
            "request_date": self.request_date.isoformat(),
            "completed_date": self.completed_date.isoformat() if self.completed_date else None
        }

class DataRemovalTracker:
    def __init__(self):
        self.requests: List[RemovalRequest] = []

    def add_request(self, site: str, url: str):
        request = RemovalRequest(
            site=site,
            url=url,
            status="pending",
            request_date=datetime.datetime.now()
        )
        self.requests.append(request)
        self.save()

    def mark_complete(self, site: str):
        for request in self.requests:
            if request.site == site and request.status == "pending":
                request.status = "completed"
                request.completed_date = datetime.datetime.now()
                self.save()
                break

    def save(self):
        with open("removal_requests.json", "w") as f:
            json.dump([r.to_dict() for r in self.requests], f, indent=2)

# Usage
tracker = DataRemovalTracker()
tracker.add_request("TruePeopleSearch", "https://www.truepeoplesearch.com/remove")
tracker.add_request("Instant Checkmate", "https://www.instantcheckmate.com/opt-out")
tracker.add_request("Radaris", "https://radaris.com/page/remove")
```

This script creates a JSON-based tracking system for managing multiple removal requests. You can extend it with automated email notifications using the `smtplib` library to remind yourself to verify completed removals.

## Using Batch Removal Services

For users with extensive digital footprints, manual removal across hundreds of data broker sites becomes impractical. Several services automate this process:

- **DeleteMe**: Subscribers pay for ongoing removal from data broker lists
- **PrivacyDuck**: Manual and automated removal with monthly reporting
- **OneRep**: Uses automation to find and remove listings across multiple brokers

While these services require subscription fees, they handle the tedious process of monitoring and re-removing your data as brokers repopulate their databases.

## Verifying Your Removal

After submitting opt-out requests, verify effectiveness through periodic checks:

```bash
# Check if your data appears on a site
curl -s "https://www.truepeoplesearch.com/results?name=YourName&city=YourCity" | grep -i "yourphone" && echo "Still indexed" || echo "Removed"

# Use Google Alerts to monitor new appearances
# Set up at: https://www.google.com/alerts
# Monitor: "Your Name" "Your City" "Your Phone Number"
```

Create a simple cron job to run these checks weekly:

```bash
# Add to crontab (crontab -e)
0 9 * * 0 /path/to/check_removal.sh >> /var/log/data_removal.log 2>&1
```

## Data Broker Removal Service Comparison

| Service | Price | Removal Speed | Monitoring | Best For |
|---------|-------|---|---|---|
| DeleteMe | $159/yr | 1 month | Monthly | coverage |
| OneRep | $8.25/mo | 2-4 weeks | Automatic re-checks | Budget-conscious users |
| PrivacyDuck | $15/mo | 2-3 weeks | Quarterly reporting | Detailed transparency |
| Incogni | $99/yr | 3-5 days | Continuous monitoring | Speed priority |

## Advanced Removal Automation with API Calls

For developers managing removal across dozens of data brokers, use API-based approaches where available:

```python
#!/usr/bin/env python3
# Automated data broker removal with retry logic

import requests
import json
import time
from typing import List, Dict, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class RemovalRequest:
    broker_name: str
    removal_url: str
    method: str  # 'POST', 'GET', 'FORM'
    request_body: Optional[Dict]
    status: str
    timestamp: datetime

class DataBrokerRemovalEngine:
    def __init__(self, firstname: str, lastname: str, email: str):
        self.firstname = firstname
        self.lastname = lastname
        self.email = email
        self.removal_requests: List[RemovalRequest] = []

        # Define removal endpoints (some brokers have undocumented APIs)
        self.brokers = {
            'peoplesoft': {
                'url': 'https://www.peoplesoft.com/remove',
                'method': 'POST',
                'params': {'firstName': firstname, 'lastName': lastname}
            },
            'intelius': {
                'url': 'https://www.intelius.com/privacy',
                'method': 'FORM',
                'form_action': 'https://www.intelius.com/privacy/remove'
            },
            'beenverified': {
                'url': 'https://www.beenverified.com/privacy/',
                'method': 'FORM',
                'form_fields': ['first_name', 'last_name', 'email']
            },
            'spokeo': {
                'url': 'https://www.spokeo.com/optout',
                'method': 'FORM',
                'requires_mobile': True
            },
            'datacheck': {
                'url': 'https://www.datacheck.com/privacy',
                'method': 'EMAIL',
                'contact': 'privacy@datacheck.com'
            }
        }

        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)'
        })

    def submit_removal_all_brokers(self) -> Dict[str, str]:
        """Attempt removal from all configured brokers"""
        results = {}

        for broker_name, broker_config in self.brokers.items():
            try:
                if broker_config['method'] == 'EMAIL':
                    result = self._send_removal_email(broker_name, broker_config)
                elif broker_config['method'] == 'POST':
                    result = self._post_removal_request(broker_name, broker_config)
                elif broker_config['method'] == 'FORM':
                    result = self._submit_html_form(broker_name, broker_config)

                results[broker_name] = result
                self._log_request(broker_name, 'submitted', result)

            except Exception as e:
                results[broker_name] = f'Failed: {str(e)}'
                self._log_request(broker_name, 'failed', str(e))

            # Respectful rate limiting
            time.sleep(2)

        return results

    def _post_removal_request(self, broker: str, config: Dict) -> str:
        """Send POST request for removal"""
        payload = config.get('params', {})

        try:
            response = self.session.post(
                config['url'],
                json=payload,
                timeout=10
            )

            if response.status_code in [200, 201, 204]:
                return f'Success: {response.status_code}'
            else:
                return f'Failed: {response.status_code}'

        except requests.exceptions.RequestException as e:
            return f'Connection error: {str(e)}'

    def _submit_html_form(self, broker: str, config: Dict) -> str:
        """Handle removal through HTML form submission"""
        try:
            # Fetch the form first
            response = self.session.get(config['url'], timeout=10)

            if response.status_code != 200:
                return f'Form fetch failed: {response.status_code}'

            # Extract CSRF token if present
            import re
            csrf_match = re.search(r'<input[^>]*name=["\']_token["\'][^>]*value=["\']([^"\']+)["\']', response.text)
            csrf_token = csrf_match.group(1) if csrf_match else None

            # Build form data
            form_data = {
                'first_name': self.firstname,
                'last_name': self.lastname,
                'email': self.email
            }

            if csrf_token:
                form_data['_token'] = csrf_token

            # Submit the form
            action_url = config.get('form_action', config['url'])
            submit_response = self.session.post(
                action_url,
                data=form_data,
                timeout=10
            )

            if submit_response.status_code in [200, 201, 204]:
                return f'Form submitted: {submit_response.status_code}'
            else:
                return f'Form submission failed: {submit_response.status_code}'

        except Exception as e:
            return f'Form error: {str(e)}'

    def _send_removal_email(self, broker: str, config: Dict) -> str:
        """Log email-based removal for manual completion"""
        return f'Manual action required: Email {config["contact"]} with removal request'

    def _log_request(self, broker: str, status: str, message: str):
        """Track all removal requests for follow-up"""
        request = RemovalRequest(
            broker_name=broker,
            removal_url=self.brokers[broker].get('url', 'N/A'),
            method=self.brokers[broker].get('method', 'UNKNOWN'),
            request_body=None,
            status=status,
            timestamp=datetime.now()
        )
        self.removal_requests.append(request)

    def generate_removal_report(self) -> str:
        """Create a report of all removal attempts"""
        report = f"Data Removal Report for {self.firstname} {self.lastname}\n"
        report += f"Generated: {datetime.now().isoformat()}\n"
        report += "=" * 60 + "\n\n"

        successful = []
        failed = []
        manual = []

        for req in self.removal_requests:
            if 'Manual' in req.status:
                manual.append(req)
            elif 'Success' in req.status:
                successful.append(req)
            else:
                failed.append(req)

        report += f"SUCCESSFUL: {len(successful)} brokers\n"
        for req in successful:
            report += f"  ✓ {req.broker_name}: {req.status}\n"

        report += f"\nFAILED: {len(failed)} brokers\n"
        for req in failed:
            report += f"  ✗ {req.broker_name}: {req.status}\n"

        report += f"\nMANUAL ACTION REQUIRED: {len(manual)} brokers\n"
        for req in manual:
            report += f"  ⚠ {req.broker_name}: {req.status}\n"

        return report

    def check_removal_status_periodic(self, broker_names: List[str], days_until_check: int = 7):
        """Schedule removal verification after time has passed"""
        verification_schedule = {}

        for broker in broker_names:
            check_date = datetime.now() + timedelta(days=days_until_check)
            verification_schedule[broker] = {
                'scheduled_check': check_date.isoformat(),
                'search_url': self._get_search_url(broker),
                'search_terms': f'{self.firstname} {self.lastname}'
            }

        return verification_schedule

    def _get_search_url(self, broker: str) -> str:
        """Get the search URL for manual verification"""
        search_urls = {
            'truepeoplesearch': 'https://www.truepeoplesearch.com/results',
            'instantcheckmate': 'https://www.instantcheckmate.com/results',
            'radaris': 'https://radaris.com/p',
            'spokeo': 'https://www.spokeo.com/search'
        }
        return search_urls.get(broker, 'N/A')

# Usage example
if __name__ == '__main__':
    # Initialize removal engine
    removal = DataBrokerRemovalEngine(
        firstname='Jane',
        lastname='Doe',
        email='jane.doe@example.com'
    )

    # Submit removal requests to all brokers
    print("Submitting removal requests...")
    results = removal.submit_removal_all_brokers()

    for broker, status in results.items():
        print(f"{broker}: {status}")

    # Generate removal report
    print("\n" + removal.generate_removal_report())

    # Schedule future verification checks
    print("\nVerification checks scheduled:")
    schedule = removal.check_removal_status_periodic(
        ['truepeoplesearch', 'instantcheckmate', 'spokeo'],
        days_until_check=14
    )
    for broker, details in schedule.items():
        print(f"  {broker}: {details['scheduled_check']}")
```

## Browser-Based Removal Workflow

Use Selenium to automate removal on sites with interactive interfaces:

```python
#!/usr/bin/env python3
# Browser-based removal automation for JavaScript-heavy sites

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
import time

class BrowserBasedRemovalBot:
    def __init__(self):
        # Configure headless Chrome
        chrome_options = Options()
        chrome_options.add_argument('--headless=new')  # New headless mode
        chrome_options.add_argument('--no-sandbox')
        chrome_options.add_argument('--disable-dev-shm-usage')
        chrome_options.add_argument('user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64)')

        self.driver = webdriver.Chrome(options=chrome_options)
        self.wait = WebDriverWait(self.driver, 10)

    def remove_from_truthfinder(self, firstname: str, lastname: str):
        """Remove from TruthFinder"""
        self.driver.get('https://www.truthfinder.com/privacy/')
        time.sleep(2)

        try:
            # Click "Remove My Record" button
            remove_btn = self.wait.until(
                EC.element_to_be_clickable((By.XPATH, "//a[contains(text(), 'Remove My Record')]"))
            )
            remove_btn.click()

            # Switch to new window if opened
            self.driver.switch_to.window(self.driver.window_handles[-1])
            time.sleep(2)

            # Fill in form
            first_input = self.driver.find_element(By.NAME, 'firstName')
            last_input = self.driver.find_element(By.NAME, 'lastName')

            first_input.send_keys(firstname)
            last_input.send_keys(lastname)

            # Submit
            submit_btn = self.driver.find_element(By.XPATH, "//button[@type='submit']")
            submit_btn.click()

            time.sleep(3)
            return {'status': 'submitted', 'site': 'TruthFinder'}

        except Exception as e:
            return {'status': 'failed', 'site': 'TruthFinder', 'error': str(e)}

    def remove_from_whitepages(self, phone: str):
        """Remove from WhitePages using phone number"""
        self.driver.get('https://support.whitepages.com/hc/en-us/articles/115015606107-How-do-I-remove-my-information-from-WhitePages')
        time.sleep(2)

        try:
            # Find removal form
            phone_input = self.wait.until(
                EC.presence_of_element_located((By.NAME, 'phone'))
            )
            phone_input.send_keys(phone)

            # Click search
            search_btn = self.driver.find_element(By.XPATH, "//button[contains(text(), 'Search')]")
            search_btn.click()

            time.sleep(3)

            # Click remove for each result
            remove_buttons = self.driver.find_elements(By.XPATH, "//button[contains(text(), 'Remove')]")
            for btn in remove_buttons:
                btn.click()
                time.sleep(1)

            return {'status': 'completed', 'site': 'WhitePages', 'removed': len(remove_buttons)}

        except Exception as e:
            return {'status': 'failed', 'site': 'WhitePages', 'error': str(e)}

    def cleanup(self):
        self.driver.quit()

# Usage
bot = BrowserBasedRemovalBot()

results = [
    bot.remove_from_truthfinder('Jane', 'Doe'),
    bot.remove_from_whitepages('555-123-4567')
]

for result in results:
    print(f"{result['site']}: {result['status']}")

bot.cleanup()
```

## Legal Rights for Data Removal

In many jurisdictions, you have statutory rights to demand data removal:

```python
# Track which removal requests are legally mandated

JURISDICTIONAL_REMOVAL_RIGHTS = {
    'US-CA': {
        'law': 'California Consumer Privacy Act (CCPA)',
        'deadline_days': 45,
        'extendable': True,
        'requires_verification': True,
        'exceptions': ['fraud prevention', 'security', 'legal obligations']
    },
    'US-VA': {
        'law': 'Virginia Consumer Data Protection Act (VCDPA)',
        'deadline_days': 45,
        'extendable': True,
        'requires_verification': True,
        'exceptions': ['fraud prevention', 'security', 'legal obligations']
    },
    'EU': {
        'law': 'GDPR Right to Erasure (Article 17)',
        'deadline_days': 30,
        'extendable': True,
        'requires_verification': True,
        'exceptions': ['legal obligation', 'public interest', 'data minimization']
    },
    'UK': {
        'law': 'UK GDPR Right to Erasure',
        'deadline_days': 30,
        'extendable': False,
        'requires_verification': True,
        'exceptions': ['legal obligation', 'public interest']
    },
    'CA': {
        'law': 'PIPEDA/CCPA Right to Erasure',
        'deadline_days': 30,
        'extendable': True,
        'requires_verification': False,
        'exceptions': ['legal records retention', 'disputes']
    }
}

def send_legal_removal_request(firstname: str, lastname: str, email: str, jurisdiction: str):
    """Generate a legally binding removal request letter"""

    jurisdiction_info = JURISDICTIONAL_REMOVAL_RIGHTS.get(jurisdiction, {})

    letter = f"""
Dear Data Protection Officer,

This is a formal request for deletion of personal data under {jurisdiction_info.get('law', 'applicable law')}.

Personal Details:
Name: {firstname} {lastname}
Email: {email}
Jurisdiction: {jurisdiction}
Request Date: {datetime.now().isoformat()}

I request immediate deletion of all personal data associated with the above information from your database.
This request includes but is not limited to:
- Name and address information
- Phone numbers and email addresses
- Historical records
- Any derived data or profiles
- Backup copies maintained for less than the statutory retention period

I am entitled to this deletion under my statutory rights. Please acknowledge receipt within {jurisdiction_info.get('deadline_days', 30)} days
and confirm deletion completion within the legal deadline.

Sincerely,
{firstname} {lastname}
{email}
"""

    return letter

# Usage
letter = send_legal_removal_request(
    firstname='Jane',
    lastname='Doe',
    email='jane@example.com',
    jurisdiction='US-CA'
)

print(letter)
```

## Additional Privacy Measures

Removing your data from people search sites addresses immediate visibility but doesn't prevent future data collection. Consider these complementary practices:

- **Limit social media exposure**: Set profiles to private and avoid posting personal details
- **Use opt-out services proactively**: Many companies have privacy request forms—use them
- **Monitor your digital footprint regularly**: Set up Google Alerts for your name and contact information
- **Request data deletion under privacy laws**: CCPA (California) and GDPR (EU) provide legal frameworks for data removal requests
- **Freeze your credit**: Contact Equifax, Experian, and TransUnion to freeze credit reports, preventing data brokers from accessing financial information
- **Register with the National Do Not Call Registry**: While limited in scope, it reduces some telemarketing use of your data


## Frequently Asked Questions


**How long does it take to remove yourself from true people search instant?**

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

- [How To Disappear From People Search Sites Complete Removal G](/privacy-tools-guide/how-to-disappear-from-people-search-sites-complete-removal-g/)
- [People Search Sites Opt Out Complete Guide 2026](/privacy-tools-guide/people-search-sites-opt-out-complete-guide-2026/)
- [What To Do If Your Personal Data Appears On People Search](/privacy-tools-guide/what-to-do-if-your-personal-data-appears-on-people-search/)
- [Facial Recognition Search Opt Out How To Remove Your Face Fr](/privacy-tools-guide/facial-recognition-search-opt-out-how-to-remove-your-face-fr/)
- [How to remove yourself from data broker sites step by step.](/privacy-tools-guide/how-to-remove-yourself-from-data-broker-sites-step-by-step-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
