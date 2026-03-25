---
layout: default
title: "Setting Up Grafana Loki for Security Logs"
description: "Deploy Grafana Loki to aggregate, query, and alert on security logs from multiple servers using Promtail and LogQL threat detection"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-grafana-loki-for-security-logs/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

Loki is a horizontally scalable log aggregation system from Grafana Labs that indexes labels rather than full log content. making it far cheaper to run than Elasticsearch for pure log storage. Combined with Promtail for log shipping and Grafana for visualization, it gives you a full security logging stack for a fraction of the cost of an ELK setup.

Architecture

```
Servers (auth.log, syslog, app logs)
     Promtail (log shipper, runs on each server)
             Loki (storage + query engine)
                     Grafana (dashboards + alerts)
```

Prerequisites

- One server to run Loki and Grafana (4GB RAM minimum for production)
- Promtail on each server you want to collect logs from
- Docker Compose (simplest deployment method)

Step 1 - Deploy Loki and Grafana with Docker Compose

```bash
mkdir -p /opt/loki && cd /opt/loki
mkdir -p data/loki data/grafana
chown 10001:10001 data/loki
```

Create `/opt/loki/docker-compose.yml`:

```yaml
version: "3.8"

services:
  loki:
    image: grafana/loki:2.9.8
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/config.yml:ro
      - ./data/loki:/loki
    command: -config.file=/etc/loki/config.yml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.4.2
    ports:
      - "3000:3000"
    volumes:
      - ./data/grafana:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=changeme
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://grafana.example.com
    restart: unless-stopped
    depends_on:
      - loki
```

Create `/opt/loki/loki-config.yml`:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  retention_delete_delay: 2h
```

Start the stack:
```bash
cd /opt/loki
docker compose up -d
docker compose logs -f
```

Step 2 - Install Promtail on Log Sources

On each server whose logs you want to ship:

```bash
Download Promtail binary
curl -LO "https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64.zip"
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

Create `/etc/promtail/config.yml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://LOKI_SERVER_IP:3100/loki/api/v1/push

scrape_configs:
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: web-01
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets: [localhost]
        labels:
          job: auth
          host: web-01
          __path__: /var/log/auth.log

  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          host: web-01
          __path__: /var/log/nginx/{access,error}.log
    pipeline_stages:
      - regex:
          expression: '^(?P<remote_addr>\S+) - (?P<user>\S+) \[(?P<timestamp>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<bytes>\d+)'
      - labels:
          status:
          method:
```

Create a systemd service:

```bash
sudo useradd -r -s /bin/false promtail
sudo mkdir -p /var/lib/promtail
sudo chown promtail:promtail /var/lib/promtail

sudo tee /etc/systemd/system/promtail.service > /dev/null <<'EOF'
[Unit]
Description=Promtail log shipper
After=network.target

[Service]
User=promtail
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
Restart=on-failure
SupplementaryGroups=adm systemd-journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now promtail
```

The `SupplementaryGroups=adm systemd-journal` gives Promtail permission to read system logs without running as root.

Step 3 - Configure Grafana Data Source

1. Open Grafana at `http://your-server:3000`
2. Log in with admin / changeme (change immediately)
3. Go to Connections > Add new data source
4. Select Loki
5. Set URL: `http://loki:3100` (or the actual IP if not using Docker networking)
6. Click Save & Test

Step 4 - Writing Security Queries with LogQL

LogQL is Loki's query language. For security monitoring:

```logql
All failed SSH login attempts
{job="auth"} |= "Failed password"

Failed logins by source IP (rate over 5 minutes)
sum by (ip) (
  rate({job="auth"} |= "Failed password"
  | regexp "from (?P<ip>[0-9.]+)"
  [5m])
)

Nginx 5xx errors in last hour
{job="nginx", status=~"5.."}

sudo usage (privilege escalation)
{job="auth"} |= "sudo"

Successful SSH logins
{job="auth"} |= "Accepted publickey" OR "Accepted password"

Top IPs hitting 404s
topk(10,
  sum by (remote_addr) (
    count_over_time({job="nginx", status="404"}[1h])
  )
)
```

Step 5 - Create Security Alerts

In Grafana, navigate to Alerting > Alert Rules > New Rule.

Alert on 5+ failed SSH attempts from the same IP in 5 minutes:

```logql
Alert query
sum by (ip) (
  rate({job="auth"} |= "Failed password"
  | regexp "from (?P<ip>[0-9.]+)"
  [5m])
) > 0.016
```

0.016/s = ~5 events per 5-minute window. Configure the alert to fire when this threshold is exceeded and send to a notification channel (Slack, email, PagerDuty).

Step 6 - Securing Loki

By default Loki has no authentication. Add basic auth via Nginx or Caddy reverse proxy:

```
Caddyfile snippet
loki.internal.example.com {
    basicauth {
        promtail $2a$14$...bcrypt-hash-of-promtail-password...
    }
    reverse_proxy localhost:3100
}
```

Then update Promtail config:
```yaml
clients:
  - url: https://loki.internal.example.com/loki/api/v1/push
    basic_auth:
      username: promtail
      password: your-promtail-password
```

Step 7 - Log Retention and Storage Management

Loki's compactor handles retention automatically if configured. Check current storage usage:

```bash
Check how much disk Loki is using
du -sh /opt/loki/data/loki/chunks/
du -sh /opt/loki/data/loki/

Check Loki's internal stats via API
curl -s http://localhost:3100/loki/api/v1/status/buildinfo | jq .
curl -s http://localhost:3100/metrics | grep loki_ingester_chunks_stored_total
```

To adjust retention without redeploying, update `loki-config.yml` and restart:

```yaml
limits_config:
  retention_period: 90d   # Change from 30d to 90d
  max_query_lookback: 90d # Must match retention_period
```

For production systems generating high log volumes, enable chunk compression:

```yaml
common:
  storage:
    filesystem:
      chunks_directory: /loki/chunks
  compactor:
    working_directory: /loki/compactor
    compaction_interval: 10m
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150

chunk_store_config:
  chunk_cache_config:
    embedded_cache:
      enabled: true
      max_size_mb: 500
```

---

Step 8 - High-Value Security Dashboards

Import or build these dashboards in Grafana to make the security data actionable:

SSH Brute Force Dashboard panels:

```logql
Panel 1 - Failed SSH attempts per minute (rate graph)
sum(rate({job="auth"} |= "Failed password" [1m])) by (host)

Panel 2 - Top attacking IPs (table)
topk(20,
  sum by (src_ip) (
    count_over_time(
      {job="auth"} |= "Failed password"
      | regexp "from (?P<src_ip>[\\d.]+)"
      [24h]
    )
  )
)

Panel 3 - Successful logins after failures (potential compromise indicator)
{job="auth"} |= "Accepted" | line_format "{{.message}}"
```

Web Application Attack Dashboard panels:

```logql
HTTP 400-499 rate (client errors including scanner noise)
sum(rate({job="nginx", status=~"4.."}[5m])) by (host)

Top paths receiving 404s (scanner enumeration detection)
topk(10,
  sum by (path) (
    count_over_time({job="nginx", status="404"}[1h])
  )
)

POST requests with 200 response to unusual paths
{job="nginx", method="POST", status="200"}
| line_format "{{.path}}"
| regexp "^(?!.*(login|api|upload|submit))(?P<suspicious_path>.+)"
```

To import a dashboard JSON:
1. Grafana → Dashboards → Import
2. Paste a dashboard JSON or enter a Grafana.com dashboard ID (e.g., 15141 for Loki logs overview)

---

Related Reading

- [How to Set Up Promtail for Log Shipping](/how-to-set-up-promtail-for-log-shipping/)
- [How to Use syslog-ng for Centralized Logging](/how-to-use-syslog-ng-for-centralized-logging/)
- [How to Monitor File Changes with inotifywait](/how-to-monitor-file-changes-with-inotifywait/)

- [Setting Up ClamAV Antivirus on Linux](/clamav-antivirus-linux-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

Related Articles

- [How to Set Up Promtail for Log Shipping](/how-to-set-up-promtail-for-log-shipping/)
- [What VPN Logs Actually Mean No Log Policy Explained](/what-vpn-logs-actually-mean-no-log-policy-explained-technically/)
- [What VPN Logs Actually Mean: No-Log Policy Explained](/what-vpn-logs-actually-mean-no-log-policy-explained-technically/)
- [Opt Out of Data Sharing Under Connecticut Data Privacy Act](/how-to-opt-out-of-data-sharing-under-connecticut-data-privac/)
- [Cloud Storage Security Breach History: Compromised](/cloud-storage-security-breach-history-compromised-services-t/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
