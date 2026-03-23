---
layout: default
title: "Self-Hosted Password Manager Comparison"
description: "Compare Vaultwarden, KeePass, Passbolt, and Padloc for self-hosting. Covers setup complexity, sync, team features, and security model."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: self-hosted-password-manager-comparison
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Self-hosting a password manager keeps your credentials off someone else's server. The tradeoff is that you own the operational burden — backups, updates, uptime. This comparison covers the four most practical options: Vaultwarden, KeePass/KeePassXC, Passbolt, and Padloc.

## Table of Contents

- [The Four Options at a Glance](#the-four-options-at-a-glance)
- [Option 1: Vaultwarden](#option-1-vaultwarden)
- [Option 2: KeePassXC](#option-2-keepassxc)
- [Option 3: Passbolt](#option-3-passbolt)
- [Option 4: Padloc](#option-4-padloc)
- [Security Considerations for All Options](#security-considerations-for-all-options)
- [Which to Choose](#which-to-choose)

## The Four Options at a Glance

| | Vaultwarden | KeePassXC | Passbolt | Padloc |
|---|---|---|---|---|
| Server required | Yes | No (file-based) | Yes | Yes |
| Team/sharing | Yes | Manual | Yes (built for teams) | Yes |
| Mobile app | Bitwarden app | KeePass apps | Official app | Official app |
| Browser extension | Bitwarden ext. | KeePassXC-Browser | Official ext. | Official ext. |
| Emergency access | Yes (via Bitwarden) | No | No | No |
| Setup difficulty | Medium | Low | Medium-High | Medium |
| License | GPLv3 | GPL | AGPLv3 | GPLv3 |

## Option 1: Vaultwarden

Vaultwarden is an unofficial Bitwarden-compatible server written in Rust. It's the most popular self-hosted option because it unlocks all Bitwarden premium features (TOTP, encrypted attachments, organizations) without a subscription.

**Pros:**
- Full compatibility with the Bitwarden ecosystem (apps, browser extensions)
- TOTP codes, secure notes, encrypted file attachments
- Organization and collection sharing
- Emergency access feature
- Active development, small memory footprint

**Cons:**
- "Unofficial" — not endorsed by Bitwarden Inc.
- You're responsible for uptime and backups
- TLS setup required (Caddy or nginx reverse proxy recommended)

### Quick Setup with Docker

```bash
docker run -d \
  --name vaultwarden \
  -e DOMAIN="https://vault.yourdomain.com" \
  -e SIGNUPS_ALLOWED=false \
  -e ADMIN_TOKEN=$(openssl rand -base64 48) \
  -v /opt/vaultwarden/data:/data \
  -p 8080:80 \
  --restart unless-stopped \
  vaultwarden/server:latest
```

Pair with Caddy for automatic HTTPS:

```
vault.yourdomain.com {
    reverse_proxy localhost:8080
}
```

After first-user signup, set `SIGNUPS_ALLOWED=false` and `INVITATIONS_ALLOWED=false` to prevent public registration.

**Best for:** Individuals or small families who want the full Bitwarden feature set without a subscription, and are comfortable managing a VPS or home server.

## Option 2: KeePassXC

KeePassXC is a local database encrypted with AES-256 or ChaCha20. There is no server — the `.kdbx` file is your vault. You sync it yourself via cloud storage, Syncthing, or a network share.

**Pros:**
- No server to manage — zero attack surface from network exposure
- Runs entirely offline if you want
- Free and open source with an active community
- Supports TOTP, SSH agent integration, browser extension
- Cross-platform (Linux, macOS, Windows)

**Cons:**
- Sync is manual — you manage it (Syncthing, Nextcloud, Dropbox)
- Conflict resolution is painful if two devices edit simultaneously
- No built-in sharing for teams
- Mobile experience depends on third-party apps (KeePass2Android, Strongbox)

### Database Setup

```bash
# Install
sudo apt install keepassxc   # Debian/Ubuntu
brew install keepassxc       # macOS

# Create a new database via GUI or CLI
keepassxc-cli db-create --set-password vault.kdbx

# Add an entry
keepassxc-cli add vault.kdbx "GitHub" --username myuser --generate-password

# Get an entry (prompts for DB password)
keepassxc-cli show vault.kdbx "GitHub"
```

**Sync with Syncthing (recommended):**

Install Syncthing on both devices and share the folder containing `vault.kdbx`. Set conflict resolution to "keep both" — KeePassXC handles merge conflicts with its own mechanism.

**Best for:** Security-conscious individuals who prefer zero network exposure and are comfortable with manual sync. Also good for air-gapped environments.

## Option 3: Passbolt

Passbolt is built for teams. It uses OpenPGP for end-to-end encryption — each password is encrypted with the public keys of people who have access. Even the server admin can't read your passwords.

**Pros:**
- Zero-knowledge architecture (server never sees plaintext)
- Built-in team permission model (groups, roles, sharing)
- CLI tool for automation and scripts
- Active open source development
- LDAP/Active Directory sync in the community edition

**Cons:**
- Docker setup is more involved than Vaultwarden
- Mobile app is basic compared to Bitwarden
- OpenPGP key management is an extra operational concept
- Performance can be slow on large vaults

### Docker Compose Setup

```yaml
version: "3.8"
services:
  passbolt:
    image: passbolt/passbolt:latest-ce
    restart: unless-stopped
    depends_on:
      - db
    environment:
      APP_FULL_BASE_URL: https://passbolt.yourdomain.com
      DATASOURCES_DEFAULT_HOST: db
      DATASOURCES_DEFAULT_DATABASE: passbolt
      DATASOURCES_DEFAULT_USERNAME: passbolt
      DATASOURCES_DEFAULT_PASSWORD: strongpassword
      EMAIL_TRANSPORT_DEFAULT_HOST: your-smtp-host
      EMAIL_DEFAULT_FROM: no-reply@yourdomain.com
    volumes:
      - gpg_volume:/etc/passbolt/gpg
      - jwt_volume:/etc/passbolt/jwt
    ports:
      - "8080:80"
      - "8443:443"

  db:
    image: mariadb:10.11
    environment:
      MYSQL_DATABASE: passbolt
      MYSQL_USER: passbolt
      MYSQL_PASSWORD: strongpassword
      MYSQL_RANDOM_ROOT_PASSWORD: "true"
    volumes:
      - database_volume:/var/lib/mysql

volumes:
  database_volume:
  gpg_volume:
  jwt_volume:
```

Create the first admin after startup:

```bash
docker compose exec passbolt su -m -c "/var/www/passbolt/bin/cake \
  passbolt register_user \
  -u admin@yourdomain.com \
  -f Admin \
  -l User \
  -r admin" -s /bin/sh www-data
```

**Best for:** Small development teams or organizations that need proper sharing controls and can manage a more complex setup.

## Option 4: Padloc

Padloc is a newer option with a clean UI and E2E encryption. The server is optional — the client runs in the browser or as a desktop app with local storage.

**Pros:**
- Very clean UI
- E2E encrypted, server never sees plaintext
- Can run fully locally without a server
- Organization support for teams

**Cons:**
- Smaller community than Vaultwarden or KeePassXC
- Fewer integrations (no SSH agent, limited browser extension)
- Self-hosted server setup is less documented

### Docker Setup

```bash
docker run -d \
  --name padloc \
  -e PL_PWA_URL=https://padloc.yourdomain.com \
  -e PL_EMAIL_SERVER=smtp.yourdomain.com \
  -e PL_EMAIL_PORT=587 \
  -e PL_EMAIL_USER=noreply@yourdomain.com \
  -e PL_EMAIL_PASSWORD=yoursmtppassword \
  -v /opt/padloc/data:/data \
  -p 3000:3000 \
  padloc/server:latest
```

**Best for:** Users who want a modern interface and are fine with a smaller ecosystem.

## Security Considerations for All Options

Regardless of which option you pick:

- **Backups**: Schedule automated backups of your vault/database to a separate encrypted location. Test restores quarterly.
- **Updates**: Self-hosted means you manage patches. Set up unattended security updates or at minimum check monthly.
- **Network exposure**: If your server is internet-facing, put it behind a reverse proxy with TLS. Consider restricting access by IP or requiring VPN.
- **Audit logs**: Passbolt and Vaultwarden both provide audit logs for organization access. Enable them.

## Which to Choose

- **Solo user, maximum simplicity**: KeePassXC with Syncthing
- **Solo/family with Bitwarden app familiarity**: Vaultwarden
- **Small team with sharing needs**: Vaultwarden or Passbolt
- **Security-first team with OpenPGP comfort**: Passbolt
- **Air-gapped or offline-only**: KeePassXC

## Frequently Asked Questions

**Can I use the first tool and the second tool together?**

Yes, many users run both tools simultaneously. the first tool and the second tool serve different strengths, so combining them can cover more use cases than relying on either one alone. Start with whichever matches your most frequent task, then add the other when you hit its limits.

**Which is better for beginners, the first tool or the second tool?**

It depends on your background. the first tool tends to work well if you prefer a guided experience, while the second tool gives more control for users comfortable with configuration. Try the free tier or trial of each before committing to a paid plan.

**Is the first tool or the second tool more expensive?**

Pricing varies by tier and usage patterns. Both offer free or trial options to start. Check their current pricing pages for the latest plans, since AI tool pricing changes frequently. Factor in your actual usage volume when comparing costs.

**How often do the first tool and the second tool update their features?**

Both tools release updates regularly, often monthly or more frequently. Feature sets and capabilities change fast in this space. Check each tool's changelog or blog for the latest additions before making a decision based on any specific feature.

**What happens to my data when using the first tool or the second tool?**

Review each tool's privacy policy and terms of service carefully. Most AI tools process your input on their servers, and policies on data retention and training usage vary. If you work with sensitive or proprietary content, look for options to opt out of data collection or use enterprise tiers with stronger privacy guarantees.

## Related Articles

- [How to Set Up Self-Hosted Password Manager Vaultwarden 2026](/how-to-set-up-self-hosted-password-manager-vaultwarden-2026/)
- [Bitwarden vs Vaultwarden Self-Hosted: A Technical Comparison](/bitwarden-vs-vaultwarden-self-hosted-comparison/)
- [Privacy Focused Password Managers Comparison 2026](/privacy-focused-password-managers-comparison-2026/)
- [Dashlane Vs 1password Comparison 2026](/dashlane-vs-1password-comparison-2026/)
- [Best Password Manager for Developers: A Technical Guide](/best-password-manager-for-developers/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
