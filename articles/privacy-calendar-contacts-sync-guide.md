---
layout: default
title: "Privacy-Focused Calendar and Contacts Sync"
description: "How to sync calendars and contacts without Google or Apple using Nextcloud, Radicale, and DAVx5 on Android and Linux desktop clients"
date: 2026-03-22
author: theluckystrike
permalink: /privacy-calendar-contacts-sync-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}


Google Calendar and Apple iCloud see everything you schedule — meetings, medical appointments, travel. The CalDAV and CardDAV protocols are open standards that let you self-host sync with any client. This guide walks through running your own calendar/contacts server and connecting it to Android, Linux, and macOS.

---

## Why Standard Protocols Matter

CalDAV (calendar) and CardDAV (contacts) are RFC-standard protocols that every major client supports. Once you have a CalDAV/CardDAV server, you can connect:

- Android: DAVx5
- iOS: built-in Contacts/Calendar app
- Linux: GNOME Calendar, Thunderbird, Evolution
- macOS: built-in Calendar/Contacts
- Windows: Thunderbird

Your data stays on your server. No vendor lock-in.

---

## Option 1: Nextcloud (Full-Featured)

Nextcloud includes CalDAV/CardDAV support with a web UI, sharing, and multi-user management. Best if you want more than just calendar/contacts.

```bash
# Install Nextcloud via Docker (simplest path)
mkdir -p ~/nextcloud/{db,html}
cat > ~/nextcloud/docker-compose.yml << 'EOF'
version: '3'
services:
  db:
    image: mariadb:10.11
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: changeme_root
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: changeme_nc
    volumes:
      - ~/nextcloud/db:/var/lib/mysql

  app:
    image: nextcloud:28
    restart: always
    ports:
      - "8080:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: changeme_nc
    volumes:
      - ~/nextcloud/html:/var/www/html
    depends_on:
      - db
EOF

cd ~/nextcloud && docker compose up -d
```

After startup, open `http://localhost:8080` and complete the setup wizard. The CalDAV URL will be:

```
https://yourdomain.com/remote.php/dav/principals/users/USERNAME/
```

---

## Option 2: Radicale (Lightweight, Single-User)

Radicale is a minimal CalDAV/CardDAV server — one Python process, no database, files stored as-is. Good for personal use on a Raspberry Pi or VPS.

```bash
# Install
pip install radicale

# Create config directory
mkdir -p ~/.config/radicale

cat > ~/.config/radicale/config << 'EOF'
[server]
hosts = 127.0.0.1:5232

[auth]
type = htpasswd
htpasswd_filename = /home/user/.config/radicale/users
htpasswd_encryption = bcrypt

[storage]
filesystem_folder = /home/user/.local/share/radicale

[logging]
level = warning
EOF

# Create user account
htpasswd -B -c ~/.config/radicale/users yourname
# Enter password when prompted

# Start Radicale
python -m radicale
```

**Run as a systemd service:**

```ini
# /etc/systemd/system/radicale.service
[Unit]
Description=Radicale CalDAV/CardDAV server
After=network.target

[Service]
User=youruser
ExecStart=/usr/local/bin/python -m radicale
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now radicale
```

**Expose via nginx reverse proxy with TLS** (required for mobile clients):

```nginx
server {
    listen 443 ssl;
    server_name cal.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/cal.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cal.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:5232;
        proxy_set_header X-Script-Name /;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass_header Authorization;
    }
}
```

---

## Connect Android with DAVx5

DAVx5 is the standard open-source CalDAV/CardDAV client for Android, available on F-Droid (no Google tracking) and Play Store.

```
Install DAVx5 from F-Droid: https://f-droid.org/packages/at.bitfire.davdroid/
```

**Setup:**

1. Open DAVx5 → **Add account**
2. Select **Login with URL and user name**
3. Base URL:
 - Nextcloud: `https://cloud.yourdomain.com/remote.php/dav/`
 - Radicale: `https://cal.yourdomain.com/`
4. Username and password as configured
5. DAVx5 auto-discovers calendars and address books
6. Select which to sync → tap **Create account**

**Sync settings:**

```
Settings → Account → Sync interval: 15 minutes
Contact group method: Groups are per-contact categories (vCard)
```

---

## Connect Linux Clients

### GNOME (Evolution Data Server)

```bash
# Install GNOME Online Accounts with CalDAV support
sudo apt install gnome-online-accounts

# Settings → Online Accounts → Add Account → Nextcloud
# Or: Settings → Online Accounts → Other (for CardDAV URL)
# Enter your server URL and credentials
```

GNOME Calendar, Contacts, and GNOME To Do all pick up accounts from Evolution Data Server automatically.

### Thunderbird + Lightning

```
Thunderbird → Calendar tab → New Calendar → On the Network
Format: CalDAV
Location: https://cal.yourdomain.com/USERNAME/calendar/
```

For contacts in Thunderbird:

```
Address Book → New Address Book → Remote Address Book
URL: https://cal.yourdomain.com/USERNAME/contacts/
```

---

## Connect macOS/iOS (Built-In)

**macOS:**

```
System Settings → Internet Accounts → Add Account → Other Account
CalDAV Account:
  Account Type: Advanced
  User Name: yourname
  Password: yourpassword
  Server Address: cal.yourdomain.com
  Server Path: /
  Port: 443
  Use SSL: checked
```

**iOS:**

```
Settings → Mail → Accounts → Add Account → Other
Add CalDAV Account → enter server, username, password
```

iOS will auto-discover calendar and contacts endpoints if your server sends the correct well-known redirects.

---

## Backup Your Calendar/Contacts Data

```bash
# Export from Radicale storage (plain vCard/iCal files)
ls ~/.local/share/radicale/collection-root/

# Nextcloud export via CLI
docker exec -u www-data nextcloud-app php occ dav:export-calendar \
  --user=yourname \
  --calendar=personal \
  --output=/tmp/personal.ics

# Backup script for Radicale
rsync -av ~/.local/share/radicale/ /backup/radicale-$(date +%Y%m%d)/
```

---

## Encryption at Rest

Radicale stores files as plain text. If the server disk is encrypted (LUKS), that covers it. For an extra layer:

```bash
# Store Radicale data inside an encrypted directory using fscrypt
fscrypt setup
fscrypt encrypt ~/.local/share/radicale/

# Or use encfs
encfs /backup/radicale-encrypted ~/.local/share/radicale/
```

Nextcloud supports server-side encryption per-file, though it adds overhead.

---

## Related Reading

- [Privacy-Focused Calendar Apps Comparison 2026](/privacy-tools-guide/privacy-focused-calendar-apps-comparison-2026/)
- [Best Encrypted Cloud Storage 2026](/privacy-tools-guide/best-encrypted-cloud-storage-2026/)
- [How to Audit Your Cloud Storage Privacy](/privacy-tools-guide/audit-cloud-storage-privacy-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)


## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.


**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


{% endraw %}
