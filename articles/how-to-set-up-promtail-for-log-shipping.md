---
layout: default
title: "How to Set Up Promtail for Log Shipping"
description: "Configure Promtail to collect, label, and ship logs from systemd, files, and syslog to Grafana Loki with pipeline transformations"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-promtail-for-log-shipping/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 6
intent-checked: true
voice-checked: true
server: http_listen_port: 9080
  grpc_listen_port: 0
  log_level: info
positions: filename: /var/lib/promtail/positions.yaml
  # Positions track where in each log file Promtail last read
  # They persist across restarts
clients: - url: http://loki-server:3100/loki/api/v1/push
    # For authenticated endpoints:
    # basic_auth:
    #   username: promtail
    #   password: secret
    # Or with TLS:
    # tls_config:
    #   ca_file: /etc/promtail/ca.crt
    #   cert_file: /etc/promtail/client.crt
    #   key_file: /etc/promtail/client.key
scrape_configs: - job_name: kubernetes-pods
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
apiVersion: apps/v1
kind: DaemonSet
metadata: annotations:
    prometheus.io/scrape: "true"
spec: selector:
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
---


## Related Articles

- [Setting Up Grafana Loki for Security Logs](/setting-up-grafana-loki-for-security-logs/)
- [How to Use AIDE for File Integrity Checking](/how-to-use-aide-for-file-integrity-checking/)
- [How to Set Up Caddy with Automatic HTTPS](/how-to-set-up-caddy-with-automatic-https/)
- [How to Use OSSEC for Host Intrusion Detection](/ossec-host-intrusion-detection-setup/)
- [How to Use Tripwire for File Integrity Monitoring](/tripwire-file-integrity-monitoring-guide/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
