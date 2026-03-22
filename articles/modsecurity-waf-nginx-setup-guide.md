---
layout: default
title: "How to Set Up ModSecurity WAF with Nginx"
description: "Install ModSecurity 3 with the OWASP Core Rule Set on Nginx to block SQLi, XSS, LFI, and RCE attacks with tuning steps to eliminate false positives"
date: 2026-03-22
author: theluckystrike
permalink: /modsecurity-waf-nginx-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# How to Set Up ModSecurity WAF with Nginx

ModSecurity is the most widely deployed open-source Web Application Firewall. Paired with the OWASP Core Rule Set (CRS), it blocks SQL injection, XSS, path traversal, remote code execution, and hundreds of other attack classes at the Nginx layer — before requests reach your application. This guide uses ModSecurity 3 (the native Nginx module) on Ubuntu 24.04.

## Architecture

```
Internet → Nginx (ModSecurity module) → Upstream App
                    ↓
           /var/log/nginx/modsec_audit.log
```

ModSecurity runs as a Nginx module. In **detection mode** it logs but doesn't block. In **enforcement mode** it returns 403 on rule matches. Always start in detection mode.

---

## 1. Install Build Dependencies

ModSecurity 3 must be compiled from source or installed from a PPA:

```bash
sudo apt update && sudo apt install -y \
  git build-essential libssl-dev libpcre3-dev \
  libyajl-dev libgeoip-dev liblmdb-dev \
  libcurl4-openssl-dev libfuzzy-dev \
  liblua5.3-dev pkg-config autoconf libtool \
  libxml2-dev libmaxminddb-dev

# For Nginx modules
sudo apt install -y nginx nginx-dev
```

---

## 2. Build ModSecurity 3

```bash
cd /tmp
git clone --depth=1 https://github.com/SpiderLabs/ModSecurity.git
cd ModSecurity
git submodule init && git submodule update

./build.sh
./configure
make -j$(nproc)
sudo make install

# Verify
ls /usr/local/modsecurity/lib/
```

---

## 3. Build the Nginx Connector Module

```bash
# Get your exact Nginx version
nginx -v  # e.g., nginx/1.24.0

cd /tmp
wget https://nginx.org/download/nginx-1.24.0.tar.gz
tar xzf nginx-1.24.0.tar.gz

git clone --depth=1 https://github.com/SpiderLabs/ModSecurity-nginx.git

cd /tmp/nginx-1.24.0
./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
make modules
sudo cp objs/ngx_http_modsecurity_module.so /usr/lib/nginx/modules/
```

---

## 4. Enable Module in Nginx

Add to the top of `/etc/nginx/nginx.conf` (outside http block):

```nginx
load_module modules/ngx_http_modsecurity_module.so;
```

---

## 5. Install OWASP Core Rule Set

```bash
sudo mkdir -p /etc/nginx/modsecurity/crs
cd /etc/nginx/modsecurity

# Download ModSecurity recommended config
sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended \
  -O modsecurity.conf

# Default is DetectionOnly — change to enforcement after tuning:
# sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' modsecurity.conf

# Download CRS
sudo git clone --depth=1 https://github.com/coreruleset/coreruleset.git crs

# Copy CRS config
sudo cp crs/crs-setup.conf.example crs/crs-setup.conf
```

Create the main WAF config that ties everything together:

```bash
sudo tee /etc/nginx/modsecurity/main.conf > /dev/null <<'EOF'
# ModSecurity config
Include /etc/nginx/modsecurity/modsecurity.conf

# Unicode mapping
SecUnicodeMapFile /etc/nginx/modsecurity/unicode.mapping 20127

# CRS setup
Include /etc/nginx/modsecurity/crs/crs-setup.conf

# CRS rules
Include /etc/nginx/modsecurity/crs/rules/*.conf

# Local exclusions (created in tuning step)
Include /etc/nginx/modsecurity/custom-exclusions.conf
EOF

# Create empty exclusions file
sudo touch /etc/nginx/modsecurity/custom-exclusions.conf

# Get unicode mapping file
sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/unicode.mapping \
  -O /etc/nginx/modsecurity/unicode.mapping
```

---

## 6. Configure Nginx Virtual Host

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    # Enable ModSecurity for this vhost
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity/main.conf;

    # Audit log
    access_log  /var/log/nginx/app_access.log;
    error_log   /var/log/nginx/app_error.log warn;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 7. Verify ModSecurity is Active

```bash
# Test SQLi detection — should log a warning (not block yet in DetectionOnly mode)
curl -s "http://localhost/?id=1'+OR+'1'='1"

# Check audit log for the event
sudo tail -f /var/log/nginx/modsec_audit.log | grep "SQL"
```

---

## 8. Switch to Enforcement Mode

Before going to enforcement, run your application through its full test suite in DetectionOnly mode and collect false positives. Then enable enforcement:

```bash
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' \
  /etc/nginx/modsecurity/modsecurity.conf

sudo systemctl reload nginx
```

Test that a real attack is now blocked:

```bash
# This should return HTTP 403
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost/?q=<script>alert(1)</script>"
# → 403
```

---

## 9. Tune False Positives

False positives are rules that block legitimate requests. Find them in the audit log:

```bash
# Parse blocked requests from audit log
sudo grep "id \"" /var/log/nginx/modsec_audit.log \
  | grep -oP 'id "\K[0-9]+' | sort | uniq -c | sort -rn | head -20
```

Disable a specific rule for a specific URI in `custom-exclusions.conf`:

```apache
# Disable rule 941100 (XSS) for the rich text editor endpoint
SecRuleUpdateTargetById 941100 "!ARGS:content"

# Disable a rule entirely for a specific path
SecRule REQUEST_URI "@beginsWith /api/admin/import" \
  "id:9000100, phase:1, pass, nolog, ctl:ruleRemoveById=949110"

# Allow a specific parameter that looks like SQLi (e.g., a report filter)
SecRuleUpdateTargetById 942100 "!ARGS:sql_filter"
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 10. Configure Anomaly Scoring (CRS v3+)

CRS uses an anomaly score threshold rather than blocking on individual rules. Each rule match adds to the score; when it exceeds the threshold, the request is blocked.

```apache
# In crs/crs-setup.conf
# Inbound anomaly threshold: 5 = strict, 10 = moderate, 20 = lenient
SecAction \
  "id:900110, \
   phase:1, \
   nolog, \
   pass, \
   t:none, \
   setvar:tx.inbound_anomaly_score_threshold=10, \
   setvar:tx.outbound_anomaly_score_threshold=4"
```

Lower the threshold gradually after testing. Start at 20 and work down to 5 over several weeks.

---

## 11. Block Bad User Agents

Add to `custom-exclusions.conf` (or a dedicated file):

```apache
# Block common scanners and exploit tools
SecRule REQUEST_HEADERS:User-Agent \
  "@pmFromFile /etc/nginx/modsecurity/bad-user-agents.txt" \
  "id:9000200, phase:1, deny, status:403, log, \
   msg:'Blocked scanner user agent'"
```

```bash
# bad-user-agents.txt (partial list)
sudo tee /etc/nginx/modsecurity/bad-user-agents.txt > /dev/null <<'EOF'
sqlmap
nikto
nessus
masscan
zgrab
python-requests/2
Go-http-client/1.1
libwww-perl
EOF
```

---

## Log Format for SIEM Integration

```nginx
# In nginx.conf log_format block
log_format modsec '$remote_addr - $remote_user [$time_local] '
                  '"$request" $status $body_bytes_sent '
                  '"$http_referer" "$http_user_agent" '
                  'modsec_status="$upstream_http_x_modsec_status"';
```

Forward `/var/log/nginx/modsec_audit.log` to Wazuh or Splunk using Filebeat with the nginx module.

---

## Related Reading

- [How to Set Up CrowdSec for Server Security](/privacy-tools-guide/crowdsec-server-security-setup-guide/)
- [How to Set Up Snort IDS on Linux](/privacy-tools-guide/snort-ids-linux-setup-guide/)
- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [How to Set Up age Encryption for Teams](/privacy-tools-guide/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
