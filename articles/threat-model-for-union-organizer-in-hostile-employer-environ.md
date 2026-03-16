---
layout: default
title: "Threat Model for Union Organizer in Hostile Employer Environment Guide"
description: "A practical security guide for union organizers facing surveillance, intimidation, or retaliation from employers. Learn threat modeling, digital security practices, and operational security for organizing campaigns."
date: 2026-03-16
author: theluckystrike
permalink: /threat-model-for-union-organizer-in-hostile-employer-environ/
categories: [guides]
---

{% raw %}

Union organizers face unique security challenges that differ significantly from typical privacy or corporate security scenarios. When employers actively monitor, intimidate, or retaliate against organizing efforts, understanding your threat model becomes essential for protecting yourself, your colleagues, and the campaign itself. This guide provides a practical framework for assessing risks and implementing appropriate countermeasures.

## Understanding the Threat Landscape

Hostile employer environments present adversaries with significant resources—corporate surveillance technology, legal teams, private investigators, and control over workplace infrastructure. Your threat model must account for these capabilities while remaining practical enough for real-world organizing work.

The first step involves identifying specific threats rather than abstract concerns. Consider three categories: technical threats (digital surveillance, device confiscation), physical threats (workplace monitoring, investigator infiltration), and legal threats (union-busting litigation, retaliatory termination). Each category requires different mitigation strategies.

A union organizer at a logistics company, for example, faced management installing keystroke logging software on work computers after rumors of organizing began circulating. Another organizer discovered their employer had purchased mobile device location data from data brokers to track workers attending union meetings. These scenarios demonstrate why threat modeling must address both obvious and less visible attack vectors.

## Digital Security Foundations

### Device Security

Your primary devices—phones, computers, and communication platforms—represent both essential tools and potential vulnerabilities. Assume that employer-controlled devices have limited privacy guarantees. Work computers, phones on corporate networks, and accounts registered with work email address expose significant data to employer surveillance.

For sensitive organizing work, maintain separate devices from your personal and work contexts. Use a dedicated phone with a privacy-focused OS for union-related communications. This separation creates legal and practical distance between your organizing activities and employer-controlled infrastructure.

```bash
# Verify your phone's network connections on iOS
# Settings > Cellular > Cellular Data Options > Cellular Data Network

# On Android, check for unusual apps with:
adb shell pm list packages -3 | grep -v google
```

Enable full-disk encryption on all devices. Modern operating systems enable this by default, but verify encryption status in your device settings. For additional protection, consider hardware encryption keys for sensitive communications.

### Communication Channels

Choose communication platforms based on their security properties and the sensitivity of your discussions. End-to-end encrypted messaging provides strong protection against interception but requires proper implementation.

Signal remains the standard for secure mobile messaging. Enable disappearing messages for sensitive conversations:

```
Signal Settings > Privacy > Disappearing Messages > After 1 hour
```

For group coordination, consider using Signal groups rather than workplace messaging platforms. Create a new group specifically for organizing work, avoiding adding members who have work devices or accounts.

Email encryption using PGP or S/MIME protects communications in transit but introduces usability challenges. Many organizers find that secure messaging apps provide better practical security through ease of use rather than relying on technically complex email encryption that users frequently misconfigure.

## Operational Security Practices

### Information Compartmentalization

Not every organizer needs access to every piece of information about the campaign. Compartmentalizing information limits the damage from any single compromise—a device being seized, an account being hacked, or an individual being pressured by management.

Establish clear boundaries: some workers handle workplace outreach, others manage legal strategy, and others coordinate with allied organizations. Each role accesses only the information necessary for their function.

When sharing sensitive documents, avoid sending them through email or cloud storage that ties to your employer. Use encrypted file transfer services or USB drives for passing sensitive materials in person.

### Device Handling During Work Hours

Workplace surveillance extends beyond digital monitoring. Employers may request or require that employees install mobile device management (MDM) profiles on personal phones used at work, granting significant surveillance capabilities.

If asked to install MDM software on your personal device, consult with an employment attorney first—in many jurisdictions, this request has legal limitations. Separating work and personal devices eliminates this threat entirely.

During union meetings or sensitive conversations, leave devices in a shielded location. Faraday bags provide modest protection against remote exploitation, though sophisticated adversaries can compromise devices through other vectors.

## Physical Security Considerations

### Workplace Surveillance

Modern workplaces often feature extensive physical surveillance systems. Understand what your employer can legally monitor: video cameras in public areas, badge swipe logs for building access, and computer activity monitoring on work devices.

Document existing surveillance as part of your threat assessment. Note camera locations, access control systems, and any monitoring you're aware of. This information helps you identify safe spaces for conversations and understand what information your employer can collect.

### Investigator Infiltration

Employers frequently hire union-busting consulting firms that deploy infiltrators into organizing campaigns. These individuals may pose as sympathetic co-workers, allied community members, or even friends of friends. Red flags include excessive interest in identifying other organizers, pressure to share internal discussions, or attempts to gather information about strategy.

Build relationships with organizers through in-person interactions before digital communication. Verify new supporters through trusted personal connections. When someone new joins the campaign, allow time for relationship-building before sharing sensitive information.

## Incident Response Planning

Prepare for potential security incidents before they occur. Develop a plan for different scenarios: a device being confiscated, an account being compromised, or an organizer being confronted by management.

### If Your Device Is Confiscated

Immediately change passwords for all sensitive accounts from a separate device. Enable two-factor authentication on accounts that didn't previously have it. Contact allies to warn them that your device may have been compromised.

For work devices, understand your legal rights regarding employer search of personal devices. In many cases, employers cannot search personal devices without consent—though they may face pressure to terminate employees who refuse.

### If Your Account Is Compromised

Assume any communication through the compromised channel is exposed. Immediately warn affected parties through alternative channels. Review account access logs to understand what information was viewed or exfiltrated.

```bash
# Review recent account access (example for Google)
# Visit: myaccount.google.com/signin/revocations
# Review security event history
```

## Building Security Culture

Effective security in organizing campaigns requires buy-in from all participants, not just technical experts. Workers who aren't security-conscious can inadvertently expose sensitive information through casual conversations, unsecured devices, or naive responses to investigator approaches.

Conduct basic security training for all organizers. Cover topics including: recognizing social engineering attempts, secure communication practices, and incident reporting procedures. Make security guidance practical rather than theoretical—explain not just what to do but why specific threats matter.

Regular security check-ins help maintain awareness. Brief discussions at organizing meetings about recent threats, new precautions, or suspicious activity keep security present without becoming burdensome.

## Conclusion

Threat modeling for union organizing requires balancing security with accessibility. Overly complex security practices discourage participation; inadequate protection exposes organizers to real risks. Start with fundamental practices—separate devices for organizing work, use encrypted messaging, and maintain information compartmentalization. Add layers of security as threats require and as organizers become comfortable with basic practices.

The goal isn't perfect security, which is impossible, but rather making targeted surveillance and intimidation sufficiently difficult that employers must resort to visible, legally risky tactics that mobilize rather than suppress organizing efforts.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
