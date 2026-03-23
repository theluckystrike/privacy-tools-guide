---
layout: default
title: "How to Monitor File Changes with inotifywait"
description: "Use inotifywait to watch directories for unauthorized changes, trigger alerts on suspicious file writes, and build automated security responses"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-monitor-file-changes-with-inotifywait/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

`inotifywait` uses the Linux kernel's inotify API to watch files and directories for changes in real time. Unlike polling-based tools, it receives kernel events instantly with zero CPU overhead when idle. This makes it ideal for security monitoring, automated backup triggers, and detecting unauthorized file modifications.

Installation

```bash
sudo apt install inotify-tools
inotifywait --version
```

Basic Usage

```bash
Watch a single file
inotifywait -m /etc/passwd

Watch a directory recursively
inotifywait -m -r /var/www/html

Watch for specific events only
inotifywait -m -e modify,create,delete,moved_from,moved_to /etc/
```

Common event types: `access`, `modify`, `attrib`, `create`, `delete`, `moved_from`, `moved_to`, `close_write`.

Formatted Output

```bash
Human-readable with timestamps
inotifywait -m -r \
  --format '%T %w %f %e' \
  --timefmt '%Y-%m-%d %H:%M:%S' \
  /etc/ /var/www/

CSV format
inotifywait -m -r \
  --csv \
  --format '%T,%w,%f,%e' \
  --timefmt '%Y-%m-%dT%H:%M:%S' \
  /home/
```

Script: Alert on Unauthorized Changes

```bash
#!/bin/bash
/usr/local/bin/watch-critical-dirs.sh

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

Script: Detect Webshell Drops

```bash
#!/bin/bash
/usr/local/bin/watch-webroot.sh

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

Systemd Service

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

Watching for Specific Patterns

```bash
Alert on new executables in temp directories
inotifywait -m -r \
  --format '%w%f' \
  -e create \
  /tmp /var/tmp /dev/shm | \
while read -r filepath; do
    if [ -x "$filepath" ]; then
        logger -p auth.crit "EXECUTABLE IN TEMP: $filepath"
    fi
done

Watch for crontab modifications (persistence mechanism)
inotifywait -m \
  -e modify,create \
  /etc/crontab /etc/cron.d/ /var/spool/cron/ 2>/dev/null | \
while read -r path event file; do
    logger -p auth.warning "CRONTAB MODIFIED: ${path}${file} ($event)"
done
```

inotifywait vs AIDE

inotifywait gives real-time alerting for immediate response. AIDE provides forensic baseline comparison that detects rootkit-level changes. Use both: inotifywait for rapid alerting, AIDE for periodic deep verification.

Advanced Monitoring Patterns

Detecting Supply Chain Attacks via File Integrity

Supply chain attacks often modify trusted binaries. Detect them by watching for unexpected changes to system executables:

```bash
#!/bin/bash
/usr/local/bin/detect-supply-chain-changes.sh

CRITICAL_BINARIES=(
  "/usr/bin/sudo"
  "/usr/bin/ssh"
  "/usr/bin/curl"
  "/usr/bin/wget"
  "/bin/bash"
  "/bin/sh"
)

BASELINE_DIR="/var/lib/file-baseline"
mkdir -p "$BASELINE_DIR"

Create baseline hashes on first run
for binary in "${CRITICAL_BINARIES[@]}"; do
  sha256sum "$binary" > "$BASELINE_DIR/$(basename $binary).hash"
done

Monitor for changes
inotifywait -m -e modify,attrib "${CRITICAL_BINARIES[@]}" | \
while read path action file; do
  current_hash=$(sha256sum "${path}${file}" | cut -d' ' -f1)
  stored_hash=$(cat "$BASELINE_DIR/$(basename ${path}${file}).hash" 2>/dev/null | cut -d' ' -f1)

  if [ "$current_hash" != "$stored_hash" ]; then
    logger -p auth.crit "SUPPLY_CHAIN_ALERT: ${path}${file} modified"
    echo "ALERT: $current_hash vs $stored_hash"

    # Quarantine immediately
    mv "${path}${file}" "${path}${file}.quarantined"
    logger -p auth.crit "Binary quarantined: ${path}${file}"
  fi
done
```

This catches scenarios where compromised package updates or repository mirrors replace binaries with malicious versions.

Detecting Privilege Escalation via setuid Changes

Attackers often create setuid binaries as persistence mechanisms. Monitor for suspicious setuid assignments:

```bash
#!/bin/bash
/usr/local/bin/detect-setuid-changes.sh

WATCH_DIRS=(
  "/usr/bin"
  "/usr/sbin"
  "/bin"
  "/sbin"
  "/usr/local/bin"
  "/home"
)

Build baseline of current setuid binaries
BASELINE="/var/lib/setuid-baseline.txt"
find "${WATCH_DIRS[@]}" -perm /4000 -type f -exec ls -lh {} \; > "$BASELINE" 2>/dev/null

log() { echo "[$(date '+%Y-%m-%dT%H:%M:%S')] $*" >> /var/log/setuid-monitor.log; }

log "Starting setuid monitoring"

inotifywait -m -r -e attrib "${WATCH_DIRS[@]}" 2>/dev/null | \
while read path action file; do
  filepath="${path}${file}"

  # Check if file now has setuid
  if [ -u "$filepath" ]; then
    # Check if it's in baseline
    if ! grep -q "$filepath" "$BASELINE" 2>/dev/null; then
      log "ALERT: New setuid binary detected: $filepath"
      ls -lh "$filepath" >> /var/log/setuid-monitor.log

      # Log to syslog
      logger -p auth.crit "SETUID_ALERT: New setuid binary: $filepath"

      # Optional: remove setuid bit immediately
      chmod u-s "$filepath"
      log "Removed setuid bit from $filepath"
    fi
  fi
done
```

Real-Time Backup Triggering

Use inotifywait to trigger backups the moment critical files change, rather than relying on scheduled backups:

```bash
#!/bin/bash
/usr/local/bin/realtime-backup.sh

CRITICAL_PATHS=(
  "/etc"
  "/var/www"
  "/srv/app"
  "/home"
)

BACKUP_DEST="/backup/realtime-$(date +%Y%m%d)"
LOG="/var/log/realtime-backup.log"

mkdir -p "$BACKUP_DEST"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG"; }

Track recently backed-up files to avoid redundant backups
BACKUP_WINDOW_SECONDS=300
declare -A last_backup_time

log "Starting real-time backup monitor"

inotifywait -m -r \
  --format '%w%f' \
  -e modify,create,delete \
  "${CRITICAL_PATHS[@]}" 2>/dev/null | \
while read filepath; do
  # Skip common temp/cache files
  case "$filepath" in
    */.git/*|*/__pycache__/*|*/.venv/*|*/node_modules/*|*/.cache/*) continue ;;
  esac

  # Debounce: don't back up same file more than once per 5 minutes
  file_key=$(echo "$filepath" | md5sum | cut -d' ' -f1)
  current_time=$(date +%s)
  last_time=${last_backup_time[$file_key]:-0}

  if [ $((current_time - last_time)) -lt $BACKUP_WINDOW_SECONDS ]; then
    continue
  fi

  last_backup_time[$file_key]=$current_time

  # Backup the file
  backup_subdir=$(dirname "$filepath" | sed 's|/|_|g')
  mkdir -p "$BACKUP_DEST/$backup_subdir"

  if [ -e "$filepath" ]; then
    cp -p "$filepath" "$BACKUP_DEST/$backup_subdir/$(basename $filepath).$(date +%s)"
    log "Backed up: $filepath"
  else
    log "Deleted (not backed up): $filepath"
  fi
done
```

Detecting Configuration Drift in Container Orchestration

For Kubernetes clusters, detect when config files drift from their committed versions:

```bash
#!/bin/bash
/usr/local/bin/detect-k8s-config-drift.sh

KUBECONFIG_DIR="/etc/kubernetes"
GIT_REPO="/var/lib/k8s-config-repo"

log() { echo "[$(date '+%Y-%m-%dT%H:%M:%S')] $*" | logger -p user.notice; }

Initial commit of configs
if [ ! -d "$GIT_REPO/.git" ]; then
  mkdir -p "$GIT_REPO"
  cd "$GIT_REPO"
  git init
  cp -r "$KUBECONFIG_DIR"/* .
  git add -A
  git commit -m "Initial K8s config baseline"
  log "Created initial K8s config baseline"
fi

Monitor for changes
inotifywait -m -r \
  --format '%w%f' \
  -e modify,create,delete \
  "$KUBECONFIG_DIR" 2>/dev/null | \
while read filepath; do
  # Sync to git repo
  relative_path=${filepath#$KUBECONFIG_DIR/}
  repo_path="$GIT_REPO/$relative_path"

  mkdir -p "$(dirname "$repo_path")"

  if [ -e "$filepath" ]; then
    cp "$filepath" "$repo_path"
  else
    rm -f "$repo_path"
  fi

  # Commit the change
  cd "$GIT_REPO"
  git add -A
  git commit -m "Config change: $relative_path" 2>/dev/null && \
    log "Config drift detected and committed: $relative_path"
done
```

Monitoring Log Files for Security Events

Watch application logs in real-time for security-relevant patterns:

```bash
#!/bin/bash
/usr/local/bin/monitor-security-logs.sh

SECURITY_PATTERNS=(
  "Failed password"
  "authentication failure"
  "DENIED"
  "SUSPICIOUS"
  "ERROR.*auth"
  "invalid user"
  "root.*attempt"
)

LOG_FILES=(
  "/var/log/auth.log"
  "/var/log/secure"
  "/var/log/audit/audit.log"
  "/var/log/apache2/error.log"
)

inotifywait -m -e modify "${LOG_FILES[@]}" 2>/dev/null | \
while read path action file; do
  # Get new lines added since last check
  tail -f "$path" 2>/dev/null &
  tail_pid=$!

  sleep 0.5

  # Check each security pattern
  for pattern in "${SECURITY_PATTERNS[@]}"; do
    if grep -i "$pattern" "$path" | tail -5 | grep -q .; then
      matching_lines=$(grep -i "$pattern" "$path" | tail -5)
      logger -p auth.warning "Security event detected in $path:
$matching_lines"
    fi
  done

  kill $tail_pid 2>/dev/null
done
```

CPU-Efficient Long-Term Monitoring

For large directory trees, inotifywait can consume CPU. Optimize with strategic watches:

```bash
#!/bin/bash
/usr/local/bin/efficient-dir-monitor.sh

Watch only directories, not every file
This drastically reduces inotify event volume

inotifywait -m -r \
  --exclude '(\.git|\.venv|node_modules|__pycache__|\.cache)' \
  --format '%w %e' \
  -e create,delete,moved_to,moved_from \
  /var/www /srv/app 2>/dev/null | \
while read dir event; do
  # Filter out noise
  case "$event" in
    CREATE|DELETE|MOVED_TO|MOVED_FROM)
      # Only process directory-level changes
      if [ -d "$dir" ]; then
        logger "Directory event: $event in $dir"
      fi
      ;;
  esac
done
```

Performance note: Watching 1 million files costs significant memory. Use `--exclude` aggressively to watch only what matters.

Related Articles

- [How to Use AIDE for File Integrity Checking](/how-to-use-aide-for-file-integrity-checking/)
- [How to Set Up Promtail for Log Shipping](/how-to-set-up-promtail-for-log-shipping/)
- [How to Use Tripwire for File Integrity Monitoring](/tripwire-file-integrity-monitoring-guide/)
- [Linux Kernel Hardening with sysctl](/linux-kernel-hardening-sysctl-guide)
- [Linux File Permissions Privacy](/linux-file-permissions-privacy-audit/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
