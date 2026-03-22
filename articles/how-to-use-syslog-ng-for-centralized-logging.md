---
layout: default
title: "How to Use syslog-ng for Centralized Logging"
description: "Set up syslog-ng to collect logs from multiple servers over TLS-encrypted connections with filtering, parsing, and structured output"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-use-syslog-ng-for-centralized-logging/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

syslog-ng is a high-performance syslog daemon that can collect, filter, parse, and forward log messages. It supports structured data, TLS transport, and flexible destination routing. This guide sets up a central log server receiving encrypted logs from multiple hosts, with filtering and JSON output.

## Installation

```bash
# Ubuntu/Debian
sudo apt install syslog-ng syslog-ng-core

# CentOS/RHEL
sudo dnf install syslog-ng

syslog-ng --version
```

## TLS Certificate Setup

```bash
mkdir -p /etc/syslog-ng/tls
cd /etc/syslog-ng/tls

# CA
openssl req -new -x509 -days 3650 -nodes \
  -keyout ca.key -out ca.crt \
  -subj "/CN=syslog-ng-CA/O=MyOrg"

# Server cert
openssl req -new -nodes \
  -keyout server.key -out server.csr \
  -subj "/CN=logserver.example.com"

openssl x509 -req -days 1825 \
  -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

# Client cert (repeat with different CN per client)
openssl req -new -nodes \
  -keyout client-web01.key -out client-web01.csr \
  -subj "/CN=web01.example.com"

openssl x509 -req -days 1825 \
  -in client-web01.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client-web01.crt

chmod 600 /etc/syslog-ng/tls/*.key
chown syslog:syslog /etc/syslog-ng/tls/*
```

## Central Server Configuration

Create `/etc/syslog-ng/syslog-ng.conf` on the log server:

```conf
@version: 4.0
@include "scl.conf"

options {
    chain_hostnames(no);
    use_dns(no);
    use_fqdn(yes);
    create_dirs(yes);
    dir_perm(0750);
};

# TLS source: receive logs from remote clients
source s_tls_clients {
    syslog(
        transport("tls")
        port(6514)
        tls(
            key-file("/etc/syslog-ng/tls/server.key")
            cert-file("/etc/syslog-ng/tls/server.crt")
            ca-dir("/etc/syslog-ng/tls/")
            peer-verify(required-trusted)
        )
    );
};

source s_local {
    system();
    internal();
};

# Destination: separate file per remote host
destination d_remote_host {
    file(
        "/var/log/remote/${HOST}/${YEAR}-${MONTH}-${DAY}.log"
        create_dirs(yes)
        owner("syslog")
        group("adm")
        perm(0640)
    );
};

# Destination: JSON output for log analysis
destination d_json {
    file(
        "/var/log/remote/all-json/${YEAR}-${MONTH}-${DAY}.json"
        create_dirs(yes)
        template("$(format-json date=${ISODATE} host=${HOST} facility=${FACILITY} severity=${LEVEL} program=${PROGRAM} pid=${PID} message=${MESSAGE})\n")
    );
};

destination d_security {
    file(
        "/var/log/remote/security/${HOST}-security.log"
        create_dirs(yes)
    );
};

# Filters
filter f_security {
    message("FAILED\|authentication failure\|Invalid user\|sudo\|su:\|pam_unix\|segfault");
};

# Log paths
log { source(s_tls_clients); destination(d_remote_host); };
log { source(s_tls_clients); destination(d_json); };
log { source(s_tls_clients); filter(f_security); destination(d_security); };
```

## Client Configuration

On each client, create `/etc/syslog-ng/conf.d/remote.conf`:

```conf
@version: 4.0

destination d_logserver {
    syslog(
        "logserver.example.com"
        port(6514)
        transport("tls")
        tls(
            key-file("/etc/syslog-ng/tls/client-web01.key")
            cert-file("/etc/syslog-ng/tls/client-web01.crt")
            ca-file("/etc/syslog-ng/tls/ca.crt")
            peer-verify(required-trusted)
        )
    );
};

log {
    source(s_src);
    destination(d_logserver);
};
```

Distribute CA cert and client key/cert to each client:

```bash
scp /etc/syslog-ng/tls/ca.crt web01:/etc/syslog-ng/tls/
scp /etc/syslog-ng/tls/client-web01.{key,crt} web01:/etc/syslog-ng/tls/
```

## Reload and Verify

```bash
# Syntax check
syslog-ng --syntax-only -f /etc/syslog-ng/syslog-ng.conf

sudo systemctl restart syslog-ng
sudo journalctl -u syslog-ng -f

# Verify logs are arriving
tail -f /var/log/remote/web01/$(date +%Y-%m-%d).log
```

## Advanced: Pattern Database

syslog-ng can classify log messages using pattern databases:

```xml
<!-- /etc/syslog-ng/patterndb.d/sshd.xml -->
<patterndb version='4' pub_date='2026-03-22'>
  <ruleset name='sshd' id='1234'>
    <patterns>
      <pattern>sshd</pattern>
    </patterns>
    <rules>
      <rule id='0001' class='security' provider='custom'>
        <patterns>
          <pattern>Failed password for @ESTRING:user: @ from @IPvANY:src_ip@ port @NUMBER:port@ @ESTRING:proto: @</pattern>
        </patterns>
        <tags>
          <tag>ssh-failed</tag>
        </tags>
      </rule>
    </rules>
  </ruleset>
</patterndb>
```

Apply in config:
```conf
parser p_db {
    db-parser(file("/etc/syslog-ng/patterndb.d/sshd.xml"));
};

log {
    source(s_tls_clients);
    parser(p_db);
    destination(d_remote_host);
};
```

## Log Retention

```bash
# /etc/logrotate.d/syslog-ng-remote
/var/log/remote/*/*.log /var/log/remote/security/*.log {
    daily
    rotate 90
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        /usr/bin/pkill -HUP syslog-ng 2>/dev/null || true
    endscript
}
```

## Related Reading

- [Setting Up Grafana Loki for Security Logs](/privacy-tools-guide/setting-up-grafana-loki-for-security-logs/)
- [How to Set Up Promtail for Log Shipping](/privacy-tools-guide/how-to-set-up-promtail-for-log-shipping/)
- [How to Use AIDE for File Integrity Checking](/privacy-tools-guide/how-to-use-aide-for-file-integrity-checking/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
