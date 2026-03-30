---
layout: default
title: "How to Set Up Promtail for Log Shipping"
description: "Configure Promtail to collect, label, and ship logs from systemd, files, and syslog to Grafana Loki with pipeline transformations"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-promtail-for-log-shipping/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Promtail is the log collection agent designed for Grafana Loki. It tails log files, attaches labels, applies pipeline transformations, and ships logs to a Loki endpoint. This guide covers installation, all common scrape configurations, pipeline stages for parsing, and TLS-secured shipping.

Installation

```bash
Check the latest release version
PROMTAIL_VERSION=$(curl -s https://api.github.com/repos/grafana/loki/releases/latest | \
  grep '"tag_name"' | cut -d'"' -f4)

Download
curl -LO "https://github.com/grafana/loki/releases/download/${PROMTAIL_VERSION}/promtail-linux-amd64.zip"
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail

Verify
promtail --version
```

Create a dedicated user and directories:
```bash
sudo useradd -r -s /bin/false promtail
sudo mkdir -p /etc/promtail /var/lib/promtail
sudo chown promtail:promtail /var/lib/promtail
```

Core Configuration Structure

Create `/etc/promtail/config.yml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: info

positions:
  filename: /var/lib/promtail/positions.yaml
  # Positions track where in each log file Promtail last read
  # They persist across restarts

clients:
  - url: http://loki-server:3100/loki/api/v1/push
    # For authenticated endpoints:
    # basic_auth:
    #   username: promtail
    #   password: secret
    # Or with TLS:
    # tls_config:
    #   ca_file: /etc/promtail/ca.crt
    #   cert_file: /etc/promtail/client.crt
    #   key_file: /etc/promtail/client.key

scrape_configs: []  # Defined in sections below
```

Scrape Config 1 - Static File Tailing

```yaml
scrape_configs:
  - job_name: nginx-access
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          env: production
          host: __HOSTNAME__
          __path__: /var/log/nginx/access.log

  - job_name: nginx-error
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-error
          host: __HOSTNAME__
          __path__: /var/log/nginx/error.log
```

The `__path__` label is special. it tells Promtail which file to tail. All other labels become queryable metadata in Loki.

Use glob patterns to tail multiple files:
```yaml
    static_configs:
      - targets: [localhost]
        labels:
          job: app-logs
          __path__: /var/log/myapp/*.log
```

Scrape Config 2 - Systemd Journal

Promtail can read directly from systemd's journal without needing log files:

```yaml
  - job_name: systemd-journal
    journal:
      max_age: 12h
      labels:
        job: systemd
        host: web-01
    relabel_configs:
      - source_labels: [__journal__systemd_unit]
        target_label: unit
      - source_labels: [__journal__hostname]
        target_label: host
      - source_labels: [__journal_priority_keyword]
        target_label: level
```

This ships all systemd unit logs. Filter to specific units via relabel rules if needed.

Scrape Config 3 - Syslog Receiver

Promtail can act as a syslog receiver (UDP/TCP):

```yaml
  - job_name: syslog-receiver
    syslog:
      listen_address: 0.0.0.0:1514
      listen_protocol: tcp
      idle_timeout: 60s
      label_structured_data: true
      labels:
        job: syslog
    relabel_configs:
      - source_labels: [__syslog_hostname]
        target_label: host
      - source_labels: [__syslog_severity]
        target_label: severity
      - source_labels: [__syslog_facility]
        target_label: facility
```

Configure remote syslog sources to forward to `promtail-host:1514`.

Pipeline Stages - Parsing and Enrichment

Pipeline stages transform log lines before shipping. They run in order:

Regex Parsing

Extract fields from log lines and promote them to labels:

```yaml
  - job_name: nginx-parsed
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - regex:
          expression: >-
            ^(?P<ip>\S+) \S+ \S+ \[(?P<timestamp>[^\]]+)\]
            "(?P<method>\S+) (?P<path>\S+) \S+"
            (?P<status>\d+) (?P<bytes>\d+)
      - labels:
          method:
          status:
      - metrics:
          http_requests_total:
            type: Counter
            description: "Total HTTP requests"
            source: status
            config:
              action: inc
```

After this pipeline, you can filter in Loki by `{status="500"}` or `{method="POST"}`.

JSON Parsing

For structured application logs:

```yaml
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: message
            user: user_id
            duration: duration_ms
      - labels:
          level:
          user:
      - drop:
          expression: ".*healthcheck.*"
          drop_counter_reason: healthcheck
```

The `drop` stage removes log lines matching a pattern before they are shipped. useful for filtering out noisy health check traffic.

Timestamp Parsing

Use the log's own timestamp rather than the ingestion time:

```yaml
    pipeline_stages:
      - json:
          expressions:
            ts: timestamp
      - timestamp:
          source: ts
          format: "2006-01-02T15:04:05.999999999Z07:00"
          # Go reference time format
```

Multiline Logs

Java stack traces and other multiline logs need to be treated as single entries:

```yaml
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
          max_wait_time: 3s
          max_lines: 128
```

Any line not starting with a date is appended to the previous line.

Systemd Service

```bash
sudo tee /etc/systemd/system/promtail.service > /dev/null <<'EOF'
[Unit]
Description=Promtail log shipper for Loki
Documentation=https://grafana.com/docs/loki/latest/send-data/promtail/
After=network-online.target
Wants=network-online.target

[Service]
User=promtail
Group=promtail
SupplementaryGroups=adm systemd-journal
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now promtail
```

Validate Configuration

```bash
Syntax check
promtail --config.file=/etc/promtail/config.yml --dry-run

Check Promtail's own HTTP endpoint for status
curl http://localhost:9080/ready
curl http://localhost:9080/metrics | grep promtail_

View targets and their status
curl http://localhost:9080/targets
```

Kubernetes Log Collection

Promtail is the standard log collector for Kubernetes clusters feeding logs to Loki. Deploy it as a DaemonSet so every node runs an agent:

```yaml
promtail-daemonset.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      tolerations:
        - operator: Exists  # Run on all nodes including control plane
      containers:
        - name: promtail
          image: grafana/promtail:2.9.8
          args:
            - -config.file=/etc/promtail/config.yml
          volumeMounts:
            - name: config
              mountPath: /etc/promtail
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: positions
              mountPath: /run/promtail
      volumes:
        - name: config
          configMap:
            name: promtail-config
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: positions
          emptyDir: {}
```

Kubernetes Promtail config with pod metadata enrichment:

```yaml
ConfigMap for Kubernetes scrape
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - cri: {}  # Parse CRI-O/containerd log format
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      # Only collect from pods that opt in
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
```

Pods opt in to log collection by adding an annotation:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
```

---

Debugging Missing Logs

```bash
Check Promtail logs
sudo journalctl -u promtail -f

Check positions file to see where Promtail is reading
cat /var/lib/promtail/positions.yaml

Verify Loki is receiving logs
curl "http://loki-server:3100/loki/api/v1/query?query=%7Bjob%3D%22nginx%22%7D&limit=5"
```

Related Reading

- [Setting Up Grafana Loki for Security Logs](/setting-up-grafana-loki-for-security-logs/)
- [How to Use syslog-ng for Centralized Logging](/how-to-use-syslog-ng-for-centralized-logging/)
- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)

- [How to Set Up age Encryption for Teams](/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
---

Related Articles

- [Setting Up Grafana Loki for Security Logs](/setting-up-grafana-loki-for-security-logs/)
- [How to Use AIDE for File Integrity Checking](/how-to-use-aide-for-file-integrity-checking/)
- [How to Set Up Caddy with Automatic HTTPS](/how-to-set-up-caddy-with-automatic-https/)
- [How to Use OSSEC for Host Intrusion Detection](/ossec-host-intrusion-detection-setup/)
- [How to Use Tripwire for File Integrity Monitoring](/tripwire-file-integrity-monitoring-guide/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
