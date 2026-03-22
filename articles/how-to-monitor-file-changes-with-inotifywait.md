---
layout: default
title: "How to Monitor File Changes with inotifywait"
description: "Use inotifywait to watch directories for unauthorized changes, trigger alerts on suspicious file writes, and build automated security responses"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-monitor-file-changes-with-inotifywait/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

`inotifywait` uses the Linux kernel's inotify API to watch files and directories for changes in real time. Unlike polling-based tools, it receives kernel events instantly with zero CPU overhead when idle. This makes it ideal for security monitoring, automated backup triggers, and detecting unauthorized file modifications.

## Installation

```bash
sudo apt install inotify-tools
inotifywait --version
```

## Basic Usage

```bash
# Watch a single file
inotifywait -m /etc/passwd

# Watch a directory recursively
inotifywait -m -r /var/www/html

# Watch for specific events only
inotifywait -m -e modify,create,delete,moved_from,moved_to /etc/
```

Common event types: `access`, `modify`, `attrib`, `create`, `delete`, `moved_from`, `moved_to`, `close_write`.

## Formatted Output

```bash
# Human-readable with timestamps
inotifywait -m -r \
  --format '%T %w %f %e' \
  --timefmt '%Y-%m-%d %H:%M:%S' \
  /etc/ /var/www/

# CSV format
inotifywait -m -r \
  --csv \
  --format '%T,%w,%f,%e' \
  --timefmt '%Y-%m-%dT%H:%M:%S' \
  /home/
```

## Script: Alert on Unauthorized Changes

```bash
#!/bin/bash
# /usr/local/bin/watch-critical-dirs.sh

WATCH_DIRS="/etc /usr/bin /usr/sbin /bin /sbin"
LOG="/var/log/file-watch.log"
ALERT_EMAIL="admin@example.com"

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $*" | tee -a "$LOG"; }

log "Starting file integrity monitor"

inotifywait -m -r \
  --format '%T %w%f %e' \
  --timefmt '%Y-%m-%dT%H:%M:%S' \
  -e modify,create,delete,attrib,moved_from,moved_to \
  $WATCH_DIRS 2>/dev/null | \
while read -r timestamp filepath event; do
    case "$filepath" in
        */proc/*|*/.git/*|*/tmp/*) continue ;;
    esac

    log "ALERT: $event on $filepath at $timestamp"

    if [[ "$filepath" == /etc/passwd || \
          "$filepath" == /etc/shadow || \
          "$filepath" == /etc/sudoers* || \
          "$filepath" =~ /usr/(s)?bin/ ]]; then
        echo "CRITICAL: $event on $filepath at $timestamp" | \
          mail -s "File Integrity Alert: $filepath" "$ALERT_EMAIL" 2>/dev/null || \
          logger -p auth.crit "FILE MONITOR: $event on $filepath"
    fi
done
```

## Script: Detect Webshell Drops

```bash
#!/bin/bash
# /usr/local/bin/watch-webroot.sh

WEBROOT="/var/www/html"
QUARANTINE="/var/quarantine"
LOG="/var/log/webshell-watch.log"

mkdir -p "$QUARANTINE"
log() { echo "$(date '+%Y-%m-%dT%H:%M:%S') $*" >> "$LOG"; }

inotifywait -m -r \
  --format '%w%f %e' \
  -e create,moved_to,close_write \
  "$WEBROOT" 2>/dev/null | \
while read -r filepath event; do
    case "${filepath,,}" in
        *.php|*.php5|*.phtml|*.jsp|*.asp|*.aspx|*.sh)
            if command -v clamscan &>/dev/null; then
                result=$(clamscan --quiet "$filepath" 2>&1)
                if [ $? -ne 0 ]; then
                    log "MALWARE DETECTED: $filepath"
                    mv "$filepath" "$QUARANTINE/"
                    logger -p auth.crit "WEBSHELL QUARANTINED: $filepath"
                    continue
                fi
            fi
            if [[ "$event" == *CREATE* ]]; then
                log "NEW PHP FILE: $filepath"
                logger -p auth.warning "NEW PHP FILE: $filepath"
            fi
            ;;
    esac
done
```

## Systemd Service

```bash
sudo tee /etc/systemd/system/file-watch.service > /dev/null <<'EOF'
[Unit]
Description=File integrity monitoring with inotifywait
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/watch-critical-dirs.sh
Restart=on-failure
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now file-watch
```

## Watching for Specific Patterns

```bash
# Alert on new executables in temp directories
inotifywait -m -r \
  --format '%w%f' \
  -e create \
  /tmp /var/tmp /dev/shm | \
while read -r filepath; do
    if [ -x "$filepath" ]; then
        logger -p auth.crit "EXECUTABLE IN TEMP: $filepath"
    fi
done

# Watch for crontab modifications (persistence mechanism)
inotifywait -m \
  -e modify,create \
  /etc/crontab /etc/cron.d/ /var/spool/cron/ 2>/dev/null | \
while read -r path event file; do
    logger -p auth.warning "CRONTAB MODIFIED: ${path}${file} ($event)"
done
```

## inotifywait vs AIDE

inotifywait gives real-time alerting for immediate response. AIDE provides forensic baseline comparison that detects rootkit-level changes. Use both: inotifywait for rapid alerting, AIDE for periodic deep verification.

## Related Articles

- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)
- [How to Set Up Promtail for Log Shipping](/privacy-tools-guide/how-to-set-up-promtail-for-log-shipping/)
- [How to Use Tripwire for File Integrity Monitoring](/privacy-tools-guide/tripwire-file-integrity-monitoring-guide/)
- [Linux Kernel Hardening with sysctl](/privacy-tools-guide/linux-kernel-hardening-sysctl-guide)
- [Linux File Permissions Privacy](/privacy-tools-guide/linux-file-permissions-privacy-audit/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
