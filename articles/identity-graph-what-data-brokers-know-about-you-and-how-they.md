---
layout: default
title: "Identity Graph What Data Brokers Know About You And How"
description: "Learn how data brokers build identity graphs to track your online activity, connect your records across platforms, and monetize your personal information"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /identity-graph-what-data-brokers-know-about-you-and-how-they/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Understand how data brokers build identity graphs by using deterministic matching (exact identifiers like email), probabilistic matching (statistical analysis of patterns), device fingerprinting, and location tracking to link your scattered data points into profiles, then use data removal services and compartmentalization strategies to minimize your graph's size. Your identity graph may contain demographic, financial, health, and behavioral data monetized across ad networks and data exchanges.

Data brokers are quietly collecting, analyzing, and selling your personal information. At the core of their operation is something called an identity graph, a sophisticated system that connects disparate pieces of data about you across the internet. Understanding how these graphs work is essential for anyone serious about protecting their privacy.

Table of Contents

- [What Is an Identity Graph?](#what-is-an-identity-graph)
- [How Data Brokers Connect Your Records](#how-data-brokers-connect-your-records)
- [What Data Brokers Know About You](#what-data-brokers-know-about-you)
- [Code Example: Simple Identity Resolution](#code-example-simple-identity-resolution)
- [Data Sources Feeding Identity Graphs](#data-sources-feeding-identity-graphs)
- [How Data Brokers Monetize Your Graph](#how-data-brokers-monetize-your-graph)
- [Advanced Identity Resolution Techniques](#advanced-identity-resolution-techniques)
- [Data Broker Inventory](#data-broker-inventory)
- [Monitoring and Removing Your Data](#monitoring-and-removing-your-data)
- [State Privacy Laws and Data Access Rights](#state-privacy-laws-and-data-access-rights)
- [The Economics of Identity Graphs](#the-economics-of-identity-graphs)
- [Protecting Yourself from Identity Graph Tracking](#protecting-yourself-from-identity-graph-tracking)

What Is an Identity Graph?

An identity graph is a database structure that links together various identifiers belonging to the same individual. These identifiers include:

- Email addresses
- Phone numbers
- Device fingerprints (MAC addresses, browser cookies)
- IP addresses
- Social media handles
- Physical addresses
- Purchase histories
- Browser behavior patterns

Data brokers aggregate this information from hundreds of sources, apps you use, websites you visit, loyalty programs you sign up for, public records, and data exchanges. The identity graph then stitches these fragments together to create a profile.

How Data Brokers Connect Your Records

The process of linking separate data points to a single identity relies on several techniques:

Deterministic Matching

This method uses exact identifiers to make connections. When you provide the same email address across multiple platforms, brokers instantly recognize these as belonging to the same person. This is the most reliable form of linking.

Probabilistic Matching

When exact identifiers aren't available, brokers use statistical analysis. If two devices frequently connect from the same IP address, share similar browsing patterns, or are active at similar times, the system calculates a probability that they belong to one person. High-confidence matches get added to the identity graph.

Device Fingerprinting

Every device has unique characteristics, screen resolution, installed fonts, browser extensions, and behavior patterns. Data brokers create fingerprints from these attributes to track users across websites, even when they clear cookies or use incognito mode.

Location Tracking

Your smartphone constantly broadcasts location data through GPS, cell tower pings, and WiFi connections. Brokers analyze location patterns to infer home addresses, workplaces, and daily routines, valuable signals for connecting seemingly unrelated data points.

What Data Brokers Know About You

Once connected, your identity graph may include:

Demographic Profile
- Age, gender, household income
- Education level, occupation
- Family composition and life stage

Financial Indicators
- Credit score ranges
- Purchase history and spending patterns
- Asset ownership (homeowner, vehicle)

Health Data
- Conditions and medications (from pharmacy records, app data)
- Fitness and wellness app integrations
- Health insurance information

Behavioral Insights
- Political affiliations and views
- Religious interests
- Travel patterns
- Entertainment preferences

Code Example: Simple Identity Resolution

Here's a basic example of how identity resolution works at the data level:

```python
class IdentityNode:
    def __init__(self, identifier, identifier_type):
        self.identifier = identifier
        self.identifier_type = identifier_type  # email, phone, device_id
        self.connections = []

    def connect(self, other_node):
        if other_node not in self.connections:
            self.connections.append(other_node)
            other_node.connections.append(self)

class IdentityGraph:
    def __init__(self):
        self.nodes = {}

    def add_identifier(self, identifier, id_type, linked_ids=None):
        node = IdentityNode(identifier, id_type)
        self.nodes[identifier] = node

        if linked_ids:
            for linked_id in linked_ids:
                if linked_id in self.nodes:
                    node.connect(self.nodes[linked_id])

Example usage
graph = IdentityGraph()
graph.add_identifier("john@email.com", "email", ["device_abc123"])
graph.add_identifier("device_abc123", "device_id", ["john@email.com"])
graph.add_identifier("+15551234567", "phone", ["john@email.com"])
```

This simplified example shows the basic concept, creating nodes for each identifier and linking them together. Real-world identity graphs are exponentially more complex, processing billions of connections.

Data Sources Feeding Identity Graphs

Data brokers collect identity information from surprisingly diverse sources:

Online Sources
- E-commerce platforms (purchase history, delivery addresses)
- Social media profiles (LinkedIn, Facebook, Twitter)
- Public records (voter registration, property records, court filings)
- Warranty registrations and product registrations
- Loyalty program memberships
- Email validation services

Offline Sources
- Credit card transactions at retail locations
- Utility company records
- Healthcare provider databases
- DMV records and vehicle registrations
- Insurance company records

Behavioral Sources
- Mobile app analytics (what apps you use, session lengths)
- Web tracking pixels and cookies
- Ad network impressions
- Mobile SDK data collection
- Location history aggregation

A single person can appear in hundreds of databases across these sources, each with incomplete information but collectively forming a profile.

How Data Brokers Monetize Your Graph

Once constructed, identity graphs generate revenue through multiple channels:

Marketing Lists: Companies buy lists of prospects matching specific characteristics. Target women aged 25-35 with household income >$100K? That's an identity graph filter.

Risk Assessment: Insurance companies, banks, and lending platforms buy graphs to assess fraud risk and credit worthiness.

Lead Generation: Advertisers buy verified contact information for cold outreach campaigns.

Data Enrichment: Services like Clearview AI sell access to identity graphs built from billions of facial recognition matches.

Targeted Advertising: Ad networks use graphs to identify users across multiple websites and devices for behavioral targeting.

A person's identity graph might be bought and sold dozens of times, with different companies adding their own data layers and inferences.

Advanced Identity Resolution Techniques

Data brokers use increasingly sophisticated methods beyond basic matching:

Household Inference: If two people share an IP address, appear in the same location data, or share device identifiers, brokers infer they live together and create household-level profiles that track spouses, children, and shared device usage.

Behavioral Clustering: Machine learning algorithms identify people with similar behavior patterns, websites visited, apps installed, purchase timing, and merge their profiles based on behavioral similarity rather than explicit identifiers.

Synthetic Identity Creation: Brokers sometimes create synthetic profiles by combining partial data from multiple individuals. This creates new profiles that may not correspond to any real person but still generate ad revenue.

Cross-Device Tracking: When you use your laptop, smartphone, and tablet, different tracking mechanisms operate on each device. Brokers correlate data across devices using probabilistic matching (same WiFi network, login behavior, timing patterns) to create unified profiles.

Here's a practical example of cross-device correlation:

```python
Simplified cross-device matching logic
def correlate_devices(devices, confidence_threshold=0.85):
    correlations = []

    for device_a, device_b in itertools.combinations(devices, 2):
        score = 0

        # Same WiFi networks
        if device_a.networks & device_b.networks:
            score += 0.3

        # Similar app installations
        app_similarity = jaccard_similarity(device_a.apps, device_b.apps)
        score += app_similarity * 0.25

        # Temporal proximity (used around same times)
        time_correlation = calculate_temporal_correlation(device_a, device_b)
        score += time_correlation * 0.25

        # Same user login on different devices
        if device_a.logged_in_email == device_b.logged_in_email:
            score += 0.2

        if score >= confidence_threshold:
            correlations.append((device_a, device_b, score))

    return correlations
```

Data Broker Inventory

Major identity graph companies include:

Experian/Equifax/TransUnion: Credit bureaus with extensive financial and identity data.

Acxiom: One of the largest, with data on over 700 million people. Recently acquired by Interpublic Group.

Clearview AI: Operates the largest facial recognition database, combining billions of photos from social media and public sources.

Data.com (owned by Salesforce): Business contact and employee information.

Spokeo: Aggregates public records and sells identity information for people searches.

Pipl: Aggregates identity data from public sources, particularly social media.

Monitoring and Removing Your Data

Services like DeleteMe and OneRep automate the opt-out process:

DeleteMe: Manually submits removal requests to major brokers. Cost: $99-129/year. Most thorough but slower.

OneRep: Automated scanning and removal. Cost: $99/year for initial removal, $49/year renewal.

Manual Opt-Out: Direct removal requests to individual brokers, time-intensive but free.

After removal, data often reappears within months as brokers repopulate from new sources. Consider annual re-removal or continuous monitoring subscriptions.

State Privacy Laws and Data Access Rights

Several US states provide rights to access and delete your identity graph data:

California (CCPA/CPRA): Right to access all collected data, delete collected data, and opt-out of sale. Data brokers must honor requests.

Virginia (VCDPA): Similar rights, applicable to Virginia residents.

Colorado (CPA): Right to delete personal data, though brokers can retain data for legitimate business purposes.

Connecticut (CTDPA): Right to access and delete, with exceptions for fraud prevention.

Utah (UCPA): Consumer privacy rights with business exemptions.

If you live in these states, you can send legal opt-out requests that brokers must honor. Many privacy lawyers offer template letters that brokers respect more than informal requests.

The Economics of Identity Graphs

Understanding why data brokers protect identity graphs helps you understand why removal is difficult:

A single identity graph might be worth $10-100 depending on data completeness and demographic value. For major brokers maintaining millions of profiles, identity graphs represent billion-dollar assets. They're incentivized to resist deletion and re-accumulate data.

This is why privacy protection requires ongoing effort. Deletion is a temporary intervention against a continuously operating system designed to rebuild graphs.

Protecting Yourself from Identity Graph Tracking

While you cannot completely eliminate yourself from these databases, you can reduce your exposure:

Limit Information Sharing
- Use privacy-focused browsers and search engines
- Decline unnecessary data collection prompts
- Use disposable email addresses for sign-ups
- Avoid online loyalty programs when possible

Opt-Out of Data Broker Listings
- Use services like DeleteMe or OneRep quarterly
- Submit direct opt-out requests to major data brokers
- Check privacy state laws for additional rights
- Maintain deletion request records for legal compliance

Use Privacy Tools
- VPNs to mask IP addresses (reduces IP-based geotagging)
- Browser extensions that block tracking scripts (Ublock Origin, Privacy Badger)
- Signal or other privacy-focused communication tools (limits metadata collection)
- Firefox Containers for isolated browsing sessions

Monitor Your Digital Footprint
- Regularly search for your name and email addresses
- Set up Google Alerts for your personal information and aliases
- Review app permissions on your devices (Android Settings > Apps > Permissions)
- Check background data usage (stops apps from phoning home location data)

Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

Can I trust these tools with sensitive data?

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

Related Articles

- [How to Remove Personal Data from Data Brokers 2026:](/how-to-remove-personal-data-from-data-brokers/---)
- [How to Remove Personal Information from Data Brokers 2026](/how-to-remove-personal-information-from-data-brokers-2026/)
- [How to Remove Personal Data from Data](/how-to-remove-personal-data-from-data-brokers/)
- [Opt Out of Data Sharing Under Connecticut Data Privacy Act](/how-to-opt-out-of-data-sharing-under-connecticut-data-privac/)
- [Vehicle Data Privacy Who Owns The Data Your Connected Car](/vehicle-data-privacy-who-owns-the-data-your-connected-car-co/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
