---
layout: default
title: "Nextcloud Setup Guide Raspberry Pi 2026"
description: "Running your own cloud storage service gives you complete control over your data. Nextcloud on a Raspberry Pi provides a cost-effective way to host file sync"
date: 2026-03-15
author: theluckystrike
permalink: /nextcloud-setup-guide-raspberry-pi-2026/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Running your own cloud storage service gives you complete control over your data. Nextcloud on a Raspberry Pi provides a cost-effective way to host file sync, calendar, contacts, and collaborative features without relying on third-party services. This guide covers the complete setup process using Docker, including reverse proxy configuration, performance optimizations, and security hardening.

## Prerequisites

You need a Raspberry Pi 4 or 5 with at least 4GB of RAM. A 32GB SD card works for the operating system, but your data should live on external storage—an USB SSD or HDD provides better performance and reliability than SD cards.

Install Raspberry Pi OS Lite (64-bit) and ensure your system is updated:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Enable SSH for remote management:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

## Installing Docker and Docker Compose

Docker simplifies Nextcloud installation by isolating dependencies and enabling straightforward updates. Install Docker using the convenience script:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

Install Docker Compose as a standalone binary:

```bash
sudo apt install -y libcompose-dev
# Or download directly for the latest version
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Verify the installation:

```bash
docker --version
docker-compose --version
```

## Configuring External Storage

Nextcloud needs persistent storage for data and the database. Mount your external drive and set proper permissions:

```bash
# Find your drive
lsblk -f

# Create mount point
sudo mkdir -p /mnt/nextcloud-data

# Mount (replace sda1 with your drive identifier)
sudo mount /dev/sda1 /mnt/nextcloud-data

# Set ownership
sudo chown -R www-data:www-data /mnt/nextcloud-data
```

For automatic mounting on reboot, add an entry to `/etc/fstab`:

```bash
# Get UUID
sudo blkid /dev/sda1

# Edit fstab
sudo nano /etc/fstab
# Add: UUID=your-uuid-here /mnt/nextcloud-data ext4 defaults,nofail 0 2
```

## Docker Compose Configuration

Create a directory for Nextcloud and write your compose file:

```bash
mkdir -p ~/nextcloud && cd ~/nextcloud
nano docker-compose.yml
```

Use this production-ready configuration:

```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.11
    container_name: nextcloud_db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - ./db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=strong_root_password
      - MYSQL_PASSWORD=strong_nextcloud_password
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    networks:
      - nextcloud_network

  redis:
    image: redis:7-alpine
    container_name: nextcloud_redis
    restart: unless-stopped
    networks:
      - nextcloud_network

  app:
    image: nextcloud:29-apache
    container_name: nextcloud_app
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./nextcloud:/var/www/html
      - /mnt/nextcloud-data:/var/www/html/data
    environment:
      - PHP_MEMORY_LIMIT=512M
      - PHP_UPLOAD_LIMIT=1G
      - NEXTCLOUD_TRUSTED_DOMAINS=your-domain.com 192.168.1.100
    depends_on:
      - db
      - redis
    networks:
      - nextcloud_network

networks:
  nextcloud_network:
    driver: bridge
```

Replace the passwords with strong values and update `NEXTCLOUD_TRUSTED_DOMAINS` with your domain or local IP address.

Start the containers:

```bash
docker-compose up -d
```

Check status:

```bash
docker-compose ps
docker-compose logs -f app
```

## Completing Nextcloud Setup

Access Nextcloud through your browser at `http://192.168.1.100:8080` (replace with your Pi's IP). Create an admin account with a strong password. The initial setup takes a few minutes.

After installation, install essential apps from the app store:
- **Nextcloud Photos** — media gallery
- **Calendar** and **Contacts** — synchronization
- **Onlyoffice** or **Collabora** — document editing
- **Tasks** — todo management

## Reverse Proxy with Nginx

Running Nextcloud behind a reverse proxy enables HTTPS and handles multiple services. Install Nginx:

```bash
sudo apt install -y nginx
```

Create a configuration file:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

Add this configuration:

```nginx
upstream nextcloud_backend {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name cloud.yourdomain.com;

    location / {
        proxy_pass http://nextcloud_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

For HTTPS, use Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d cloud.yourdomain.com
```

## Performance Tuning

The Raspberry Pi has limited resources. Optimize Nextcloud for better performance:

### PHP-FPM Configuration

Create a custom PHP configuration:

```bash
mkdir -p ~/nextcloud/php
nano ~/nextcloud/php/www.conf
```

Adjust these settings for the Pi's capabilities:

```ini
[www]
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 500
```

Mount this custom config in your compose file by adding to the app service:

```yaml
volumes:
  - ./php/www.conf:/usr/local/etc/php-fpm.d/zz-docker.conf
```

### Redis Caching

Redis significantly improves performance by caching data. Ensure your `config.php` includes:

```php
'memcache.distributed' => '\\OC\\Memcache\\Redis',
'memcache.local' => '\\OC\\Memcache\\Redis',
'redis' => [
    'host' => 'redis',
    'port' => 6379,
],
```

Access the config:

```bash
docker exec -it nextcloud_app bash
nano /var/www/html/config/config.php
```

### Database Optimization

Create an optimization script:

```bash
docker exec -it nextcloud_db mysql -u root -p nextcloud -e "
CREATE TABLE IF NOT EXISTS oc_migrations (
    app VARCHAR(255) NOT NULL,
    version VARCHAR(255) NOT NULL,
    PRIMARY KEY (app, version)
);
"
```

Run database optimizations periodically:

```bash
docker exec --user www-data nextcloud_app php occ db:convert-filecache-bigint
docker exec --user www-data nextcloud_app php occ maintenance:repair
```

## Security Hardening

Protect your Nextcloud instance with these measures:

### Two-Factor Authentication

Enable 2FA for all users through the admin panel. Install the Two-Factor TOTP Provider app for time-based codes.

### Fail2Ban Protection

Install Fail2Ban to block brute force attempts:

```bash
sudo apt install -y fail2ban
```

Create a Nextcloud filter:

```bash
sudo nano /etc/fail2ban/filter.d/nextcloud.conf
```

```ini
[Definition]
failregex = {"app":"core","message":"Login failed:.*","level":2,"time".*}
```

Add to Fail2Ban config:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[nextcloud]
enabled = true
port = http,https
filter = nextcloud
logpath = /var/www/nextcloud/data/nextcloud.log
maxretry = 5
bantime = 3600
```

### Regular Backups

Create a backup script:

```bash
#!/bin/bash
BACKUP_DIR="/backup/nextcloud"
DATE=$(date +%Y%m%d)

mkdir -p $BACKUP_DIR

# Backup data directory
tar -czf $BACKUP_DIR/data-$DATE.tar.gz /mnt/nextcloud-data

# Backup database
docker exec nextcloud_db mysqldump -u nextcloud -p strong_nextcloud_password nextcloud > $BACKUP_DIR/db-$DATE.sql

# Keep only last 7 days
find $BACKUP_DIR -type f -mtime +7 -delete
```

Schedule daily backups with cron:

```bash
crontab -e
# Add: 0 2 * * * /path/to/backup.sh
```

## Updating Nextcloud

Regular updates patch security vulnerabilities. Before updating, create a backup:

```bash
cd ~/nextcloud
docker-compose down
tar -czf backup-$(date +%Y%m%d).tar.gz ./
docker-compose up -d
```

Pull the latest image and recreate containers:

```bash
docker-compose pull
docker-compose up -d
docker-compose logs -f
```

Run post-update commands:

```bash
docker exec --user www-data nextcloud_app php occ maintenance:mode --on
docker exec --user www-data nextcloud_app php occ upgrade
docker exec --user www-data nextcloud_app php occ maintenance:mode --off
```

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Troubleshooting Hub](/privacy-tools-guide/troubleshooting-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
