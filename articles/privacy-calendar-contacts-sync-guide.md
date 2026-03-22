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

## Migrating from Google Calendar and Google Contacts

Before you can use your own server, you need to export your existing data. Google provides clean export tools for both services.

**Export Google Calendar:**

```
Google Calendar → Settings (gear icon) → Settings
→ Import & export → Export
Downloads a .zip containing .ics files for each calendar
```

```bash
# Extract and inspect the export
unzip calendar_export.zip -d ~/calendar_export
ls ~/calendar_export/
# You'll see files like: personal@gmail.com.ics, birthdays.ics

# Import into Radicale by copying to the collection folder
cp ~/calendar_export/personal.ics \
  ~/.local/share/radicale/collection-root/yourname/calendar.ics
```

**Export Google Contacts:**

```
Google Contacts → Export → Google CSV or vCard format
```

```bash
# Convert Google CSV to vCard if needed
pip install vobject

python3 << 'EOF'
import csv, vobject

with open('contacts.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        card = vobject.vCard()
        card.add('fn').value = row['Name']
        if row['E-mail 1 - Value']:
            email = card.add('email')
            email.value = row['E-mail 1 - Value']
        print(card.serialize())
EOF
```

For most users, exporting as vCard (`.vcf`) directly from Google Contacts is simpler — the file imports directly into Radicale or Nextcloud Contacts without conversion.

## Troubleshooting Common Sync Problems

**DAVx5 shows "Sync error" on Android:**

The most common cause is a self-signed TLS certificate. DAVx5 requires a valid certificate chain. Use Let's Encrypt:

```bash
# Install certbot and get a certificate
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d cal.yourdomain.com
# Certbot auto-configures nginx and sets up renewal
```

If you must use a self-signed certificate (internal network only), add it to Android's trusted certificates via Settings → Security → Install certificates.

**Contacts not appearing after sync:**

DAVx5 stores contacts in a separate account. The Android Contacts app must be configured to display all accounts:

```
Android Contacts → Menu → Manage contacts → Contacts to display
→ Select "All contacts" or check the DAVx5 account
```

**Calendar events disappear after editing on another client:**

This usually indicates a timezone handling mismatch. Set a consistent timezone on your server:

```bash
# For Radicale, ensure events are stored with explicit TZID
# Check a problematic event:
cat ~/.local/share/radicale/collection-root/yourname/calendar/*.ics | grep TZID

# Nextcloud: set server timezone
docker exec -u www-data nextcloud-app php occ config:system:set logtimezone --value="America/New_York"
```

**High battery drain from DAVx5:**

Reduce sync frequency for less time-sensitive data:

```
DAVx5 → Account → Calendar sync interval: 1 hour
DAVx5 → Account → Contact sync interval: 4 hours
```

Contacts change rarely — daily or even manual sync is sufficient for most users.

## Privacy Considerations Beyond the Server

Running your own CalDAV/CardDAV server eliminates Google and Apple from seeing your schedule, but other data flows still deserve attention.

**DNS leaks your calendar server address.** When DAVx5 syncs, it queries your DNS provider for `cal.yourdomain.com`. If you use your ISP's DNS or a logging resolver, your sync activity is visible. Use a privacy-respecting resolver:

```bash
# On Android: Settings → Network → Private DNS
# Enter: dns.quad9.net or 1dot1dot1dot1.cloudflare-dns.com
```

**Your VPN provider sees sync traffic timing.** Even over TLS, the timing and size patterns of CalDAV sync can reveal meeting frequency. For high-sensitivity use cases, route sync traffic over a VPN or Tor.

**Email invitations bypass your private server entirely.** Meeting invites sent via `.ics` email attachments come through your email provider. If privacy-of-meetings is critical, use a private email provider alongside your CalDAV server — Tutanota and Proton Mail both support calendar invitations independently of Google.

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

## Multi-User Setup on Radicale

Radicale supports multiple users from a single instance. Each user gets their own collection namespace, and access is controlled through htpasswd authentication.

```bash
# Add a second user to Radicale
htpasswd -B ~/.config/radicale/users seconduser
# Enter password when prompted

# Radicale automatically creates a separate namespace:
# /seconduser/calendar/
# /seconduser/contacts/
```

For per-user calendar sharing (allowing one user to read another's calendar), Radicale's rights system handles this through a `rights` config file:

```ini
# ~/.config/radicale/rights
[owner-write]
user: .+
collection: ^{user}/.*$
permissions: rw

[shared-read]
user: alice
collection: ^bob/shared-calendar/.*$
permissions: r
```

This gives `alice` read access to `bob`'s shared-calendar collection. Nextcloud has a web UI for this — useful for households or small teams who want shared family calendars without giving everyone full access.

## Encryption at Rest

Radicale stores files as plain text. If the server disk is encrypted (LUKS), that covers it. For an extra layer:

```bash
# Store Radicale data inside an encrypted directory using fscrypt
fscrypt setup
fscrypt encrypt ~/.local/share/radicale/

# Or use encfs
encfs /backup/radicale-encrypted ~/.local/share/radicale/
```

Nextcloud supports server-side encryption per-file using AES-256, though it adds CPU overhead and makes backup restoration more complex — only use it if you have a specific threat model requiring it beyond full-disk encryption.

---

## Related Reading

- [Privacy-Focused Calendar Apps Comparison 2026](/privacy-tools-guide/privacy-focused-calendar-apps-comparison-2026/)
- [Best Encrypted Cloud Storage 2026](/privacy-tools-guide/best-encrypted-cloud-storage-2026/)
- [How to Audit Your Cloud Storage Privacy](/privacy-tools-guide/audit-cloud-storage-privacy-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
