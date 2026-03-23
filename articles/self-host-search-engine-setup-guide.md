---
layout: default
title: "How to Self-Host a Privacy Search Engine"
description: "Deploy SearXNG or Whoogle as a self-hosted, privacy-respecting meta search engine that strips tracking, removes ads, and keeps your queries private"
date: 2026-03-22
author: theluckystrike
permalink: /self-host-search-engine-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

How to Self-Host a Privacy Search Engine

Every search query to Google, Bing, or DuckDuckGo is logged, profiled, and used to build a behavioral record. Self-hosting a meta search engine means your queries pass through your server to multiple search backends. no account, no persistent profile, no tracking cookie. This guide covers SearXNG and Whoogle, the two strongest self-hosted options.

SearXNG vs Whoogle

| Feature | SearXNG | Whoogle |
|---------|---------|---------|
| Source | Meta search (20+ engines) | Google proxy |
| Results quality | Broad, configurable | Google-quality |
| Setup complexity | Moderate | Simple |
| Customizable engines | Yes (per-query) | No |
| Images, videos, news | Yes | Yes |
| JavaScript required | Yes | Optional |
| Resource usage | ~200MB RAM | ~100MB RAM |

SearXNG queries Bing, DuckDuckGo, Startpage, Wikipedia, and others simultaneously and merges results. Whoogle proxies Google, so results match Google exactly. without tracking.

Option 1: SearXNG with Docker

```yaml
/opt/searxng/docker-compose.yml
version: '3.8'

services:
  searxng:
    image: searxng/searxng:latest
    restart: always
    ports:
      - 127.0.0.1:8080:8080
    volumes:
      - ./searxng:/etc/searxng:rw
    environment:
      - SEARXNG_BASE_URL=https://search.yourdomain.com/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"

  redis:
    image: redis:alpine
    restart: always
    command: redis-server --save "" --appendonly "no"
    cap_drop:
      - ALL
    cap_add:
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"
```

```bash
mkdir -p /opt/searxng/searxng
cd /opt/searxng

Generate a secret key
docker compose run --rm searxng sh -c "openssl rand -hex 32 > /etc/searxng/secret_key" 2>/dev/null

Create initial settings file
cat > searxng/settings.yml << 'EOF'
use_default_settings: true

server:
  secret_key: !include secret_key
  limiter: true
  image_proxy: true
  method: GET

search:
  safe_search: 0
  autocomplete: ""
  default_lang: "en"

ui:
  static_use_hash: true
  default_locale: "en"
  query_in_title: false
  infinite_scroll: true
  center_alignment: false
  results_on_new_tab: false
  theme_args:
    simple_style: auto

Disable engines that break frequently or require accounts
engines:
  - name: google
    disabled: true   # use Google via Startpage to avoid blocks
  - name: startpage
    disabled: false
  - name: ddg definitions
    disabled: false
  - name: duckduckgo
    disabled: false
  - name: bing
    disabled: false
  - name: wikipedia
    disabled: false
  - name: github
    disabled: false
EOF

docker compose up -d
```

SearXNG Configuration Deep Dive

```yaml
searxng/settings.yml. extended privacy configuration

server:
  secret_key: !include secret_key
  bind_address: "0.0.0.0"
  port: 8080
  limiter: true     # rate limit per IP. important for public instances
  public_instance: false  # set true if you allow public access

Outgoing requests. anonymize or route through Tor
outgoing:
  request_timeout: 3.0
  max_request_timeout: 10.0
  # Route requests through a proxy (optional)
  # proxies:
  #   http:  socks5://127.0.0.1:9050  # Tor
  #   https: socks5://127.0.0.1:9050

Privacy: do not log searches
general:
  debug: false
  donation_url: false
  contact_url: false
  enable_metrics: false  # disables the /stats page

Search result settings
search:
  safe_search: 0
  formats:
    - html
    - json    # enables the JSON API for scripts
    - csv
    - rss

Enabled engines with categories
engines:
  - name: duckduckgo
    engine: duckduckgo
    categories: [guides]
    disabled: false

  - name: startpage
    engine: startpage
    categories: general
    disabled: false
    timeout: 6.0

  - name: wikipedia
    engine: wikipedia
    categories: general
    language: en
    disabled: false

  - name: github
    engine: github
    categories: code
    disabled: false
```

Option 2: Whoogle (Google Results, No Tracking)

Whoogle is simpler. it proxies Google results through your server, stripping ads, AMP links, and tracking parameters.

```yaml
/opt/whoogle/docker-compose.yml
version: '3.8'

services:
  whoogle:
    image: benbusby/whoogle-search:latest
    restart: always
    ports:
      - 127.0.0.1:5000:5000
    environment:
      # Proxy outgoing requests through Tor (optional)
      # WHOOGLE_PROXY_LOC: socks5://127.0.0.1:9050
      # WHOOGLE_PROXY_TYPE: socks5

      WHOOGLE_CONFIG_COUNTRY: US
      WHOOGLE_CONFIG_LANGUAGE: lang_en
      WHOOGLE_CONFIG_SEARCH_LANGUAGE: lang_en
      WHOOGLE_CONFIG_SAFE: 0
      WHOOGLE_CONFIG_DARK: 1
      WHOOGLE_CONFIG_ALTS: 1    # replace YouTube/Twitter/Reddit with privacy frontends
      WHOOGLE_CONFIG_VIEW_IMAGE: 1
      WHOOGLE_CONFIG_GET_ONLY: 1
      WHOOGLE_CONFIG_URL: https://search.yourdomain.com/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
```

```bash
cd /opt/whoogle && docker compose up -d
```

Whoogle replaces links to:
- YouTube → Invidious
- Reddit → Teddit or Libreddit
- Twitter → Nitter
- Instagram → Bibliogram

Nginx Reverse Proxy

```nginx
/etc/nginx/sites-available/search
server {
    listen 443 ssl;
    server_name search.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/search.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/search.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # If private instance. restrict to your IP ranges
    # allow 10.0.0.0/8;
    # allow 192.168.0.0/16;
    # deny all;

    # Rate limit search queries
    limit_req_zone $binary_remote_addr zone=search:10m rate=30r/m;
    limit_req zone=search burst=10 nodelay;

    location / {
        proxy_pass http://127.0.0.1:8080;   # SearXNG
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Remove identifying headers upstream
        proxy_set_header X-Real-IP "";
    }
}
```

Browser Integration

Set as Default Search Engine in Firefox

1. Navigate to your SearXNG instance
2. Right-click the address bar → "Add Search Engine"
3. Go to `about:preferences#search` → set as default

Or add manually:

```javascript
// In about:config (Firefox)
// browser.urlbar.suggest.searches = false (disable search suggestions sent to provider)

// Add as OpenSearch engine (SearXNG supports this automatically)
// Visit: https://search.yourdomain.com/
// Click the + icon in the address bar
```

Using the SearXNG JSON API

```bash
Query via API
curl "https://search.yourdomain.com/search?q=linux+security&format=json" \
  | jq '.results[:3] | .[] | {title, url, content}'

Script that searches and opens the first result
#!/bin/bash
QUERY="${*// /+}"
URL=$(curl -s "https://search.yourdomain.com/search?q=${QUERY}&format=json" \
  | jq -r '.results[0].url')
echo "Opening: $URL"
xdg-open "$URL"
```

Monitoring and Maintenance

```bash
Check for updates
cd /opt/searxng && docker compose pull && docker compose up -d

Check logs for errors
docker compose logs searxng --tail 50

Monitor container health
docker stats searxng

SearXNG: review which engines are returning results
Visit: https://search.yourdomain.com/stats
(disable metrics in settings if public instance)
```

Related Articles

- [Privacy Focused Search Engines Comparison 2026](/privacy-focused-search-engines-comparison-2026/)
- [Best Privacy-Focused Search Engines Comparison 2026](/best-privacy-focused-search-engines-comparison-2026/)
- [Right To Be Forgotten In Search Engines How To Request](/right-to-be-forgotten-in-search-engines-how-to-request-googl/)
- [How to Disable Google AMP Tracking in Search Results Guide](/how-to-disable-google-amp-tracking-in-search-results-guide/)
- [Facial Recognition Search Opt Out How To Remove Your Face](/facial-recognition-search-opt-out-how-to-remove-your-face-fr/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
