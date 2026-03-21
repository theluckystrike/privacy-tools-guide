---
layout: default
title: "Set Up Catch All Email Domain For Unlimited Private"
description: "A practical guide to configuring catch-all email domains for creating unlimited private email aliases. Perfect for developers and power users who want"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-catch-all-email-domain-for-unlimited-private-aliases/
categories: [guides, security]
reviewed: true
intent-checked: true
voice-checked: true
score: 8
tags: [privacy-tools-guide]
---

{% raw %}

If you sign up for services online, your inbox probably receives a steady stream of marketing emails, data breach notifications, and promotional newsletters you never requested. A catch-all email domain lets you create unlimited email aliases on the fly, keeping your primary inbox clean and protecting your identity. This guide walks you through setting up a catch-all email domain from scratch.

## Understanding Catch-All Email Domains

A catch-all email domain (also called a wildcard domain) intercepts any email sent to addresses that don't explicitly exist on your mail server. When you configure `*@yourdomain.com` as a catch-all, emails to `amazon@yourdomain.com`, `netflix@yourdomain.com`, or any random string all land in your inbox.

For privacy-focused users, this means you can generate unique aliases for every service:

- `github@yourdomain.com` for your GitHub account
- `twitter@yourdomain.com` for Twitter
- `airbnb@yourdomain.com` for Airbnb

When any of these services gets compromised or sells your data, you immediately know which one leaked your email address. You can then block that specific alias without affecting your other accounts.

## Prerequisites

Before configuring your catch-all domain, ensure you have:

1. **A domain name** — You can buy one through Namecheap, Cloudflare, or Gandi
2. **A domain registrar** — Where you manage DNS records
3. **An email forwarding service** — To receive emails sent to your catch-all domain
4. **A primary inbox** — Where forwarded emails will arrive

## Step 1: Choose an Email Forwarding Service

Several services handle catch-all email forwarding without requiring you to run your own mail server:

- **Forward Email** (forwardemail.net) — Free tier available, open source, straightforward setup
- **Cloudflare Email Routing** — Free, handles routing directly from Cloudflare DNS
- **ImprovMX** (improvmx.com) — Free and paid tiers, simple configuration

For this guide, we'll use Cloudflare Email Routing since it's free and integrates with your existing DNS management.

## Step 2: Configure DNS Records

Assuming you already have your domain pointing to Cloudflare, here's how to set up email routing:

1. Log into your Cloudflare dashboard
2. Select your domain
3. Navigate to **Email Routing** → **Routing**
4. Click **Create email rule**
5. Under **Destination**, add your primary email address (e.g., `yourname@gmail.com`)
6. Save the rule

Cloudflare provides specific MX records you need to add. The typical values:

```
Type: MX
Name: @ or yourdomain.com
Priority: 10
Value: mx1.forwardemail.net
```

Replace the values with whatever your email forwarding service provides.

## Step 3: Set Up Catch-All Routing

With Cloudflare Email Routing, the routing rules handle catch-all functionality automatically. Any email sent to `anything@yourdomain.com` forwards to your destination address.

If you're using Forward Email, add these DNS records:

```
Type: MX
Name: @
Priority: 10
Value: mx1.forwardemail.net

Type: TXT
Name: @
Value: v=spf1 include:spf.forwardemail.net -all
```

## Step 4: Create Aliases for Specific Services

Now comes the practical part — generating aliases. There are two main approaches:

### Approach A: Service-Based Aliases

Create aliases using the service name:

```
github@yourdomain.com
aws@yourdomain.com
linkedin@yourdomain.com
```

This makes it immediately obvious which service leaked your email.

### Approach B: Random Aliases

Generate random strings for maximum privacy:

```
x7k9m2@yourdomain.com
q4rt8w@yourdomain.com
```

This approach prevents services from guessing other aliases you might use.

### Implementation with a Password Manager

Most password managers support generating email aliases directly. 1Password, Bitwarden, and Proton Pass include this functionality:

```bash
# Using 1Password CLI to generate a masked email (if configured)
op item create --title "New Service Account" \
  item.credential.username="random-string@yourdomain.com" \
  item.credential.password="$(openssl rand -base64 20)"
```

## Step 5: Automate Alias Creation

For power users, you can script alias creation using your domain provider's API:

```bash
#!/bin/bash
# generate-alias.sh - Generate a new email alias

DOMAIN="yourdomain.com"
SERVICE="$1"

if [ -z "$SERVICE" ]; then
  echo "Usage: $0 <service-name>"
  exit 1
fi

ALIAS="${SERVICE}-$(date +%Y%m%d)@${DOMAIN}"
echo "Generated alias: $ALIAS"
echo "Use this alias for $SERVICE registration"

# Optional: Log to a local file for tracking
echo "$(date): $SERVICE -> $ALIAS" >> ~/aliases.log
```

Run it like this:

```bash
./generate-alias.sh github
# Output: Generated alias: github-20260316@yourdomain.com
```

## Step 6: Manage Incoming Emails

Since catch-all domains receive all emails sent to non-existent addresses, you'll inevitably get spam. Here are strategies to manage it:

### Use Email Filters

Set up filters in your primary inbox to automatically sort forwarded emails:

```yaml
# Example Gmail filter criteria
- If email matches: to:(*@yourdomain.com)
  Then: Apply label "Aliases" and star it
```

### Create Service-Specific Rules

If you want different handling per alias, use your forwarding service's rules:

- Route `github-*@yourdomain.com` to one inbox
- Route `finance-*@yourdomain.com` to another
- Route everything else to your default inbox

## Step 7: Handle Reply functionality

Standard catch-all forwarding lets you receive emails, but replying from your alias requires more setup. Options include:

1. **Simple forwarding only** — You receive emails but reply from your primary address
2. **Email aliases with reply capability** — Services like Addy.io or SimpleLogin let you reply without revealing your primary email
3. **Custom SMTP** — Run your own mail server with postfix for full control

For most users, the forwarding-only approach works well. When you need to reply, your recipient sees your alias in the "From" header, or you can configure your email client to send from your alias address.

## Troubleshooting Common Issues

### Emails Not Being Delivered

- Verify MX records are correctly configured
- Check that your TXT record includes proper SPF authentication
- Wait 24-48 hours for DNS propagation

### Emails Marked as Spam

- Ensure your sending domain has proper SPF, DKIM, and DMARC records
- Some providers are strict about forwarded emails
- Consider using a dedicated email service with better sender reputation

### Domain Already in Use

- Some email providers block known catch-all domains
- Consider using multiple domain names for different purposes
- Use a domain primarily for email (not website hosting) to maintain better reputation

## Security Considerations

While catch-all domains provide excellent privacy, keep these points in mind:

- **Email content is still visible** to your forwarding service provider
- **Metadata leaks** — Email headers still reveal timing and routing information
- **Service limitations** — Some services block known alias domains
- **Recovery options** — Ensure your primary email is secure since it becomes the gateway

For maximum privacy, consider combining catch-all aliases with encrypted email services like ProtonMail or Tutanota.


## Related Articles

- [Set Up Mail In A Box Private Email Server Complete 2026](/privacy-tools-guide/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Best Privacy-Focused Email Aliases Service Comparison 2026](/privacy-tools-guide/best-privacy-focused-email-aliases-service-comparison-2026/)
- [How To Set Up Private Bitcoin Full Node At Home For Transact](/privacy-tools-guide/how-to-set-up-private-bitcoin-full-node-at-home-for-transact/)
- [How to Set Up Private DNS on Android for All Apps](/privacy-tools-guide/how-to-set-up-private-dns-on-android-for-all-apps/)
- [How To Set Up Self Hosted Matrix Synapse Server For Private](/privacy-tools-guide/how-to-set-up-self-hosted-matrix-synapse-server-for-private-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
