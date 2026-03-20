---

layout: default
title: "Domain Name Inheritance: How to Transfer Registrar Accounts and Website Hosting After Death"
description: "Learn how to transfer domain names and hosting accounts after death. This guide covers registrar transfer processes, documentation requirements, and technical steps for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /domain-name-inheritance-how-to-transfer-registrar-accounts-a/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
---

{% raw %}

Domain names and web hosting accounts represent valuable digital assets that require planning for transfer upon death. Unlike physical property, these assets exist in registrar databases and rely on account credentials that may not survive the account holder. This guide covers the technical and procedural steps developers and power users need to transfer domain names and hosting after death.

## Understanding Domain Name Transfer Mechanics

Domain name transfers involve moving registration from one registrar to another, or transferring control within the same registrar to a different account. The process requires authorization from the current registrant and cooperation from both the losing and gaining registrars.

ICANN, the governing body for domain names, establishes transfer policies that all accredited registrars must follow. The standard transfer process includes:

1. **Unlock the domain** at the current registrar
2. **Obtain the authorization code** (also called EPP code or transfer key)
3. **Initiate the transfer** at the new registrar
4. **Confirm via email** sent to the domain's administrative contact
5. **Wait for propagation** (typically 5-7 days)

For transfers occurring after death, registrars typically require additional documentation beyond the standard authorization code.

## Required Documentation for Death-Related Transfers

When a domain owner dies, registrars need proof of death and authority to act on behalf of the deceased. Documentation requirements vary by registrar, but expect to provide:

- **Death certificate** (certified copy, not a scan)
- **Letter of testamentary** or **letter of administration** if the estate is in probate
- **Court order** appointing the requester as personal representative or executor
- **ID verification** of the person requesting the transfer

Some registrars accept alternative documentation like:
- Beneficiary designation on file with the registrar
- Transfer-on-death (TOD) provisions registered with the domain
- Small estate affidavits for domains of limited value

Contact the registrar's legal or customer support department before gathering documents. Policy differences between registrars can affect which documents they accept.

## Step-by-Step Transfer Process

### Step 1: Locate Account Credentials

Before any transfer can occur, you need access to or information about the deceased's registrar account. This includes:

- The registrar name (GoDaddy, Namecheap, Cloudflare, etc.)
- The account email address
- Any stored payment methods or billing information

If the deceased used a password manager, check there first. Many developers store registrar credentials alongside other service credentials. If no password manager exists, the registrar may require additional identity verification.

### Step 2: Contact the Registrar

Email or call the registrar's support team explaining the situation. Use a subject line like "Death Transfer Request - [Domain Name]" and include:

- Your name and relationship to the deceased
- The domain name(s) in question
- A summary of what you're requesting
- Your contact information

Most registrars have dedicated processes for death-related transfers. GoDaddy calls this a "Trust/Estate" transfer request. Namecheap handles these through their legal department. Cloudflare requires court documentation.

### Step 3: Submit Required Documentation

After initial contact, the registrar will specify what documents they need. Send certified copies rather than originals—they typically don't return documents. Keep copies for your records.

Expect a processing time of 2-4 weeks for documentation review. Some registrars expedite this for active businesses, but estate transfers rarely receive priority treatment.

### Step 4: Complete the Technical Transfer

Once documentation clears, the registrar will guide you through the technical transfer:

**For transfers to a different registrar:**
```
1. Log into the deceased's registrar account (or have support unlock it)
2. Disable domain privacy if enabled
3. Obtain the transfer authorization code
4. Initiate transfer at the new registrar with the authorization code
5. Confirm transfer approval email from the old registrar
```

**For account-to-account transfers within the same registrar:**
```
1. Provide the gaining account ID to the registrar
2. Sign a transfer authorization form (provided by registrar)
3. Wait for registrar to complete the internal transfer
```

## Hosting Account Transfers

Web hosting accounts add complexity because they often contain:

- Website files and databases
- Email accounts and archives
- SSL certificates
- Application configurations

### Transferring cPanel/WHM Accounts

If the deceased used cPanel hosting, you have several transfer options:

**Full account transfer (recommended for complex sites):**
1. Request a full backup from the host using the deceased's account
2. Download the backup archive
3. Create a new account at your chosen host
4. Restore the backup to the new account

**Partial transfer (for simpler needs):**
1. Use FTP/SFTP to download public_html contents
2. Export databases via phpMyAdmin or command line:
   ```bash
   mysqldump -u username -p database_name > backup.sql
   ```
3. Upload to new hosting and reimport databases

### Transferring Cloud Infrastructure

For developers using AWS, Google Cloud, or similar providers:

- **AWS**: Requires a death certificate and legal documentation. The process goes through AWS Trust & Safety. You may need to create a new account and rebuild resources, as direct account transfers aren't supported.
  
- **Google Cloud**: Similar requirements. Contact Google Cloud Support with documentation proving your authority over the estate.

- **DigitalOcean/Vultr/Linode**: These providers typically allow account access transfers with proper documentation. Open a support ticket explaining the situation.

### SSL Certificate Considerations

SSL certificates are tied to specific domains and may include the organization name. When transferring:

1. Export the private key and certificate from the old server:
   ```bash
   # Apache
   cat /etc/ssl/certs/domain.crt /etc/ssl/private/domain.key
   
   # Nginx
   cat /etc/nginx/ssl/domain.crt /etc/nginx/ssl/domain.key
   ```

2. Install the certificate on the new server, or
3. Generate a new certificate using Let's Encrypt (free) on the new host:
   ```bash
   certbot certonly --webroot -w /var/www/html -d example.com
   ```

New certificates take effect immediately and avoid transfer complications with organization-validated certificates.

## Prevention: Setting Up Transfers Before Death

The easiest transfer is one planned in advance. Developers and power users can take several proactive steps:

### Enable Transfer-on-Death Provisions

Some registrars (notably GoDaddy and certain TLDs) support transfer-on-death designations. Set this up in the domain settings to bypass probate requirements.

### Document Everything

Maintain a secure document containing:

- Registrar account credentials
- Hosting provider information
- Access credentials for any related services
- Instructions for your designated executor

Store this in a password manager with emergency access enabled, or in a secure physical location known to your executor.

### Use Multi-User Access

If using services like Cloudflare or GitHub, add team members or organizations. This provides automatic access continuation without transfer processes.

### Register Domains Through a Business Entity

Registering domains through an LLC or corporation can simplify transfers if the entity survives the owner. The domain becomes an asset of the business rather than personal property.

## Summary

Transferring domain names and hosting after death requires patience and documentation. The key steps are:

1. Identify all domain and hosting accounts the deceased owned
2. Gather death certificates and legal documentation proving your authority
3. Contact each registrar and hosting provider individually
4. Complete technical transfers using authorization codes
5. Rebuild or transfer website assets to new hosting

Prevention through transfer-on-death designations, comprehensive documentation, and multi-user access eliminates most transfer headaches. Take time now to document your digital assets and designate who should manage them.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
