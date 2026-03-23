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

Installation

```bash
Ubuntu/Debian
sudo apt install syslog-ng syslog-ng-core

CentOS/RHEL
sudo dnf install syslog-ng

syslog-ng --version
```

TLS Certificate Setup

```bash
mkdir -p /etc/syslog-ng/tls
cd /etc/syslog-ng/tls

CA
openssl req -new -x509 -days 3650 -nodes \
  -keyout ca.key -out ca.crt \
  -subj "/CN=syslog-ng-CA/O=MyOrg"

Server cert
openssl req -new -nodes \
  -keyout server.key -out server.csr \
  -subj "/CN=logserver.example.com"

openssl x509 -req -days 1825 \
  -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

Client cert (repeat with different CN per client)
openssl req -new -nodes \
  -keyout client-web01.key -out client-web01.csr \
  -subj "/CN=web01.example.com"

openssl x509 -req -days 1825 \
  -in client-web01.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client-web01.crt

chmod 600 /etc/syslog-ng/tls/*.key
chown syslog:syslog /etc/syslog-ng/tls/*
```

Central Server Configuration

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

TLS source: receive logs from remote clients
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

Destination: separate file per remote host
destination d_remote_host {
    file(
        "/var/log/remote/${HOST}/${YEAR}-${MONTH}-${DAY}.log"
        create_dirs(yes)
        owner("syslog")
        group("adm")
        perm(0640)
    );
};

Destination: JSON output for log analysis
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

Filters
filter f_security {
    message("FAILED\|authentication failure\|Invalid user\|sudo\|su:\|pam_unix\|segfault");
};

Log paths
log { source(s_tls_clients); destination(d_remote_host); };
log { source(s_tls_clients); destination(d_json); };
log { source(s_tls_clients); filter(f_security); destination(d_security); };
```

Client Configuration

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

Reload and Verify

```bash
Syntax check
syslog-ng --syntax-only -f /etc/syslog-ng/syslog-ng.conf

sudo systemctl restart syslog-ng
sudo journalctl -u syslog-ng -f

Verify logs are arriving
tail -f /var/log/remote/web01/$(date +%Y-%m-%d).log
```

Advanced: Pattern Database

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

Log Retention

```bash
/etc/logrotate.d/syslog-ng-remote
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

Forwarding to Elasticsearch or Loki

syslog-ng can ship parsed logs directly to Elasticsearch or Grafana Loki for indexing and dashboards.

Elasticsearch destination:

```conf
@version: 4.0
@module elasticsearch-http

destination d_elastic {
    elasticsearch-http(
        index("syslog-{YEAR}.{MONTH}.{DAY}")
        type("")
        url("https://elastic.example.com:9200/_bulk")
        user("logstash")
        password("YOUR_PASS")
        tls(
            ca-file("/etc/syslog-ng/tls/elastic-ca.crt")
            peer-verify(required-trusted)
        )
        template("$(format-json date=${ISODATE} host=${HOST} program=${PROGRAM}
                   severity=${LEVEL} message=${MESSAGE} facility=${FACILITY})\n")
    );
};

log {
    source(s_tls_clients);
    destination(d_elastic);
};
```

Grafana Loki destination via HTTP:

```conf
destination d_loki {
    http(
        url("http://loki.example.com:3100/loki/api/v1/push")
        method("POST")
        headers("Content-Type: application/json")
        body("{ \"streams\": [{ \"stream\": { \"host\": \"${HOST}\",
              \"program\": \"${PROGRAM}\" },
              \"values\": [[\"${UNIXTIME}000000000\",
              \"${MESSAGE}\"]] }] }")
    );
};
```

Install the Elasticsearch or HTTP module if not present:

```bash
sudo apt install syslog-ng-mod-http
sudo systemctl restart syslog-ng
```

Monitoring syslog-ng Health

Track dropped messages and connection errors to catch log gaps before they become a problem:

```bash
Statistics. shows message counters per source/destination
syslog-ng-ctl stats | grep -E "(processed|dropped|queued)"

Watch for connection errors in real time
sudo journalctl -u syslog-ng -f | grep -i "error\|warn"

Check TLS handshake failures (clients with expired certs)
sudo journalctl -u syslog-ng | grep "SSL error"
```

Add a Prometheus metrics endpoint for automated monitoring:

```conf
/etc/syslog-ng/conf.d/metrics.conf
@version: 4.0
source s_metrics {
    metrics-probe();
};
```

Then scrape `localhost:8080/metrics` with your Prometheus instance. Key metrics to alert on:
- `syslogng_input_events_total` dropping to zero for a source (client stopped sending)
- `syslogng_output_events_dropped_total` rising (destination overloaded or offline)
- `syslogng_connections` for the TLS source dropping to zero (network partition or cert expiry)

Rate Limiting and Throttling Incoming Logs

High-volume log sources can overwhelm your central server. syslog-ng has built-in throttling at the source level:

```conf
Limit a single TCP source to 1000 messages per second
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
        log-msg-size(65536)
        so-rcvbuf(1048576)
    );
};

Throttle in a log path. drop excess messages rather than queue
log {
    source(s_tls_clients);
    throttle(value(1000));  # messages per second
    destination(d_remote_host);
    flags(flow-control);    # apply backpressure to sender
};
```

For destinations that are slower than the source (e.g., writing to a slow NFS mount), enable disk buffering to prevent message loss:

```conf
destination d_remote_host_buffered {
    file(
        "/var/log/remote/${HOST}/${YEAR}-${MONTH}-${DAY}.log"
        disk-buffer(
            mem-buf-size(64Mi)
            disk-buf-size(2Gi)
            reliable(yes)
            dir("/var/spool/syslog-ng")
        )
    );
};
```

The disk buffer persists to `/var/spool/syslog-ng` so no messages are lost if the destination stalls.

---

TLS Certificate Rotation Without Downtime

When TLS certificates near expiration, rotate them without interrupting log collection:

```bash
Generate new server certificate (keep the CA the same)
cd /etc/syslog-ng/tls

openssl req -new -nodes \
  -keyout server-new.key -out server-new.csr \
  -subj "/CN=logserver.example.com"

openssl x509 -req -days 1825 \
  -in server-new.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server-new.crt

Verify the new cert before deploying
openssl verify -CAfile ca.crt server-new.crt
openssl x509 -noout -dates -in server-new.crt

Swap atomically
mv server.key server-old.key && mv server.crt server-old.crt
mv server-new.key server.key && mv server-new.crt server.crt

Reload syslog-ng (no restart needed. reload re-reads TLS config)
systemctl reload syslog-ng

Verify new cert is in use
echo | openssl s_client -connect logserver.example.com:6514 2>/dev/null | \
  openssl x509 -noout -dates
```

Client certificates can be rotated independently. the CA validates them, so as long as the CA cert does not change, each client can get a new cert without touching the server.

---

Related Articles

- [How To Configure Postfix With Mandatory Tls Encryption](/how-to-configure-postfix-with-mandatory-tls-encryption-for-e/)
- [Openvpn Tls Auth Vs Tls Crypt Difference Security Comparison](/openvpn-tls-auth-vs-tls-crypt-difference-security-comparison/)
- [How to Set Up mTLS for Microservices](/mtls-microservices-setup-guide/)
- [Tls Fingerprinting How Your Browser Handshake Identifies](/tls-fingerprinting-how-your-browser-handshake-identifies-you/)
- [VPN TLS Fingerprinting: How Censors Identify VPN Protocols](/vpn-tls-fingerprinting-how-censors-identify-vpn-protocols-ex/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
