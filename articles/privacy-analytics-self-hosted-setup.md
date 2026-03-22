---
layout: default
title: "Privacy-Focused Analytics: Self-Hosted Options"
description: "Deploy Plausible, Umami, or Matomo on your own server for GDPR-compliant analytics without sending visitor data to third parties or requiring cookie banners"
date: 2026-03-22
author: theluckystrike
permalink: /privacy-analytics-self-hosted-setup/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

# Privacy-Focused Analytics: Self-Hosted Options

Self-hosting your analytics means visitor data never leaves your infrastructure. No third-party JavaScript on your pages, no cross-site tracking, no GDPR consent banners required (for cookie-free analytics), and no feeding behavioral data to ad networks. This guide covers deploying Plausible, Umami, and Matomo on your own server.

## Key Takeaways

- **No third-party JavaScript on your pages**: no cross-site tracking, no GDPR consent banners required (for cookie-free analytics), and no feeding behavioral data to ad networks.
- **Topics covered**: why self-host vs. cloud analytics, option 1: plausible (lightweight, cookie-free), option 2: umami (minimal, multi-site)
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Why Self-Host vs. Cloud Analytics

| Factor | Google Analytics | Plausible Cloud | Self-Hosted |
|--------|-----------------|-----------------|-------------|
| Data location | Google's servers | Provider's servers | Your servers |
| GDPR cookies required | Yes | No | Depends on tool |
| Third-party JS on pages | Yes | Yes | No |
| Data ownership | Google owns it | You own it | You own it |
| Cost | Free (you are the product) | $9+/mo | Server cost only |

## Option 1: Plausible (Lightweight, Cookie-Free)

Plausible is the simplest option: no cookies, no personal data, GDPR-compliant without a consent banner.

```bash
# /opt/plausible/docker-compose.yml
version: '3.8'

services:
  mail:
    image: bytemark/smtp
    restart: always

  plausible_db:
    image: postgres:14-alpine
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: postgres

  plausible_events_db:
    image: clickhouse/clickhouse-server:23.3-alpine
    restart: always
    volumes:
      - event-data:/var/lib/clickhouse
      - ./clickhouse/clickhouse-config.xml:/etc/clickhouse-server/config.d/logging.xml:ro
      - ./clickhouse/clickhouse-user-config.xml:/etc/clickhouse-server/users.d/logging.xml:ro
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

  plausible:
    image: ghcr.io/plausible/community-edition:v2.1
    restart: always
    command: sh -c "sleep 10 && /entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"
    depends_on:
      - plausible_db
      - plausible_events_db
      - mail
    ports:
      - 127.0.0.1:8000:8000
    env_file:
      - plausible-conf.env

volumes:
  db-data:
  event-data:
```

```bash
# plausible-conf.env
BASE_URL=https://analytics.yourdomain.com
SECRET_KEY_BASE=$(openssl rand -base64 48)
DATABASE_URL=postgres://postgres:postgres@plausible_db:5432/plausible_db
CLICKHOUSE_DATABASE_URL=http://plausible_events_db:8123/plausible_events_db
MAILER_EMAIL=noreply@yourdomain.com
```

```bash
mkdir -p /opt/plausible/clickhouse
cd /opt/plausible

# Create ClickHouse config files (reduces logging noise)
cat > clickhouse/clickhouse-config.xml << 'EOF'
<clickhouse><logger><level>warning</level><console>true</console></logger><query_thread_log remove="remove"/><query_log remove="remove"/><text_log remove="remove"/><trace_log remove="remove"/><metric_log remove="remove"/><asynchronous_metric_log remove="remove"/><session_log remove="remove"/><part_log remove="remove"/></clickhouse>
EOF

cat > clickhouse/clickhouse-user-config.xml << 'EOF'
<clickhouse><profiles><default><log_queries>0</log_queries><log_query_threads>0</log_query_threads></default></profiles></clickhouse>
EOF

docker compose up -d
```

```bash
# Add the tracking snippet to your site
# Replace your-domain.com with your actual domain as registered in Plausible
```

```html
<script defer data-domain="your-domain.com" src="https://analytics.yourdomain.com/js/script.js"></script>
```

## Option 2: Umami (Minimal, Multi-Site)

Umami is a clean alternative with real-time stats, multi-user support, and a simpler stack than Plausible.

```yaml
# /opt/umami/docker-compose.yml
version: '3.8'

services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    restart: always
    ports:
      - 127.0.0.1:3000:3000
    environment:
      DATABASE_URL: postgresql://umami:umami@db:5432/umami
      DATABASE_TYPE: postgresql
      APP_SECRET: $(openssl rand -hex 32)
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - umami-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  umami-db-data:
```

```bash
cd /opt/umami && docker compose up -d

# Default login: admin / umami — change immediately
# Access at localhost:3000 (put behind nginx/Traefik with TLS)
```

Umami tracking snippet:

```html
<script async src="https://analytics.yourdomain.com/script.js" data-website-id="your-website-id"></script>
```

## Option 3: Matomo (Feature-Rich, GDPR Mode)

Matomo is the closest to Google Analytics in features. It supports heatmaps, funnels, A/B testing (paid plugins), and a full GDPR consent mode.

```bash
# Install on bare metal or VM (not Docker — Matomo works better with traditional LAMP)
sudo apt install apache2 php php-mysql php-curl php-gd php-xml php-mbstring mariadb-server

# Create database
sudo mysql -e "CREATE DATABASE matomo; GRANT ALL ON matomo.* TO 'matomo'@'localhost' IDENTIFIED BY 'strong-password';"

# Download Matomo
wget https://builds.matomo.org/matomo.zip
unzip matomo.zip -d /var/www/html/
sudo chown -R www-data:www-data /var/www/html/matomo
```

### GDPR Configuration in Matomo

```bash
# Matomo can anonymize IPs by default (no consent banner needed for basic stats)
# Settings → Privacy → Anonymize visitor information:
# - Anonymize IP addresses: last 2 bytes
# - Anonymize User ID: yes
# - Force new visit on browser date change: yes
# - Delete old raw data: after 6 months

# Or use Matomo's Consent Manager plugin for full consent flow
```

## Nginx Reverse Proxy for Any of the Above

```nginx
# /etc/nginx/sites-available/analytics
server {
    listen 443 ssl;
    server_name analytics.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/analytics.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/analytics.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Block access to admin from outside your network
    location /settings {
        allow 10.0.0.0/8;
        allow 192.168.0.0/16;
        deny all;
        proxy_pass http://127.0.0.1:8000;
    }

    location / {
        # Plausible
        proxy_pass http://127.0.0.1:8000;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Cache the analytics script aggressively
        location ~* \.(js)$ {
            expires 1d;
            add_header Cache-Control "public, immutable";
            proxy_pass http://127.0.0.1:8000;
        }
    }
}
```

## Blocking Bot Traffic

All three tools have some bot filtering, but you can add your own at the nginx level:

```nginx
# Add to your analytics nginx config
map $http_user_agent $is_bot {
    default         0;
    ~*bot           1;
    ~*crawl         1;
    ~*spider        1;
    ~*slurp         1;
    ~*googlebot     1;
    ~*bingbot       1;
}

server {
    # ...
    location /api/event {
        if ($is_bot) { return 200; }
        proxy_pass http://127.0.0.1:8000;
    }
}
```

## Backup Your Analytics Data

```bash
# Plausible: backup PostgreSQL + ClickHouse
docker exec plausible_db pg_dump -U postgres plausible_db | gzip > /backup/plausible-pg-$(date +%Y%m%d).sql.gz
docker exec plausible_events_db clickhouse-client --query="SELECT * FROM plausible_events_db.events FORMAT Native" | gzip > /backup/plausible-ch-$(date +%Y%m%d).gz

# Umami: just PostgreSQL
docker exec umami-db-1 pg_dump -U umami umami | gzip > /backup/umami-$(date +%Y%m%d).sql.gz

# Matomo: MySQL + files
mysqldump -u matomo -p matomo | gzip > /backup/matomo-db-$(date +%Y%m%d).sql.gz
tar czf /backup/matomo-files-$(date +%Y%m%d).tar.gz /var/www/html/matomo/
```

## Related Reading

- [Plausible vs Matomo vs Fathom: Privacy Analytics Compared](/privacy-tools-guide/plausible-vs-matomo-vs-fathom-privacy-focused-analytics-comp/)
- [How to Set Up Privacy-Preserving Customer Analytics](/privacy-tools-guide/how-to-set-up-privacy-preserving-customer-analytics-without-/)
- [How to Configure Google Analytics Alternative for GDPR](/privacy-tools-guide/how-to-configure-google-analytics-alternative-for-gdpr-compl/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
