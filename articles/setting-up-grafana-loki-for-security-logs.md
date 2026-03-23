---
layout: default
title: "Setting Up Grafana Loki for Security Logs"
description: "Deploy Grafana Loki to aggregate, query, and alert on security logs from multiple servers using Promtail and LogQL threat detection"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-grafana-loki-for-security-logs/
categories: [guides, security]
tags: [privacy-tools-guide, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
version: "3.8"
services: loki:
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
auth_enabled: false
server: http_listen_port: 9080
  grpc_listen_port: 0
common: storage:
    filesystem:
      chunks_directory: /loki/chunks
  compactor:
    working_directory: /loki/compactor
    compaction_interval: 10m
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150
schema_config: configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
limits_config: retention_period: 90d   # Change from 30d to 90d
  max_query_lookback: 90d # Must match retention_period
compactor: working_directory: /loki/compactor
  retention_enabled: true
  retention_delete_delay: 2h
positions: filename: /var/lib/promtail/positions.yaml
clients: - url: https://loki.internal.example.com/loki/api/v1/push
    basic_auth:
      username: promtail
      password: your-promtail-password
scrape_configs: - job_name: syslog
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
  rate({job="auth"} |= "Failed password"
  | regexp "from (?P<ip>[0-9.]+)"
  [5m])
  sum by (remote_addr) (
    count_over_time({job="nginx", status="404"}[1h])
  )
Example: Alert on 5+ failed SSH attempts from the same IP in 5 minutes:
  rate({job="auth"} |= "Failed password"
  | regexp "from (?P<ip>[0-9.]+)"
  [5m])
    basicauth {
        promtail $2a$14$...bcrypt-hash-of-promtail-password...
    }
    reverse_proxy localhost:3100
chunk_store_config: chunk_cache_config:
    embedded_cache:
      enabled: true
      max_size_mb: 500
---


## Related Reading

- [How to Set Up Promtail for Log Shipping](/how-to-set-up-promtail-for-log-shipping/)
- [How to Use syslog-ng for Centralized Logging](/how-to-use-syslog-ng-for-centralized-logging/)
- [How to Monitor File Changes with inotifywait](/how-to-monitor-file-changes-with-inotifywait/)

- [Setting Up ClamAV Antivirus on Linux](/clamav-antivirus-linux-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

## Related Articles

- [How to Set Up Promtail for Log Shipping](/how-to-set-up-promtail-for-log-shipping/)
- [What VPN Logs Actually Mean No Log Policy Explained](/what-vpn-logs-actually-mean-no-log-policy-explained-technically/)
- [What VPN Logs Actually Mean: No-Log Policy Explained](/what-vpn-logs-actually-mean-no-log-policy-explained-technically/)
- [Opt Out of Data Sharing Under Connecticut Data Privacy Act](/how-to-opt-out-of-data-sharing-under-connecticut-data-privac/)
- [Cloud Storage Security Breach History: Compromised](/cloud-storage-security-breach-history-compromised-services-t/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
