---
layout: default
title: "How to Set Up mTLS for Microservices"
description: "Configure mutual TLS authentication between microservices using a private CA, cert-manager, and Envoy or nginx to prevent unauthorized service-to-servic..."
date: 2026-03-22
author: theluckystrike
permalink: /mtls-microservices-setup-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 7
intent-checked: true
voice-checked: true
---
{% raw %}

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: name: internal-ca
spec: ca:
    secretName: internal-ca-key-pair


Certificate Renewal Automation Without Downtime

Short-lived certificates (90 days or less) reduce the blast radius of a key compromise, but manual renewal causes outages. The correct pattern uses overlapping validity windows and pre-rotation.

With cert-manager's `renewBefore` field set to 15 days, the certificate rotates well before expiry. Your nginx or app server must reload the new certificate file without dropping connections:

```bash
nginx: reload config without connection drops
sudo nginx -t && sudo nginx -s reload

For Go services. watch for file changes and reload the tls.Certificate
Use fsnotify to trigger a reload of the TLS config in the HTTP server
```

For services that embed certificates in memory at startup, add a SIGHUP handler:

```go
// main.go. reload TLS cert on SIGHUP
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGHUP)
go func() {
    for range sigChan {
        newCert, err := tls.LoadX509KeyPair(certFile, keyFile)
        if err != nil {
            log.Printf("cert reload error: %v", err)
            continue
        }
        certMu.Lock()
        currentCert = &newCert
        certMu.Unlock()
        log.Println("TLS certificate reloaded")
    }
}()
```

Monitor expiry across all services proactively:

```bash
#!/bin/bash
check-cert-expiry.sh. alert on certs expiring within 14 days
for service in user-service order-service payment-service; do
    expiry=$(openssl x509 \
        -in /etc/mtls-certs/${service}/${service}.crt \
        -noout -enddate 2>/dev/null | cut -d= -f2)
    expiry_epoch=$(date -d "$expiry" +%s 2>/dev/null || date -j -f "%b %d %H:%M:%S %Y %Z" "$expiry" +%s)
    now_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - now_epoch) / 86400 ))
    if [[ $days_left -lt 14 ]]; then
        echo "WARNING: ${service} cert expires in ${days_left} days (${expiry})"
    else
        echo "OK: ${service} cert expires in ${days_left} days"
    fi
done
```

---

Related Articles

- [Mumble Encrypted Voice Chat Server Setup For Private Team](/mumble-encrypted-voice-chat-server-setup-for-private-team-co/)
- [Self-Hosted Private Git Server with Gitea](/private-git-server-gitea-setup-guide/)
- [How To Prepare Pgp Key Revocation Certificate For Publicatio](/how-to-prepare-pgp-key-revocation-certificate-for-publicatio/)
- [iCloud Private Relay: How It Works vs](/ios-private-relay-how-it-works-vs-vpn/)
- [How To Set Up Self Hosted Matrix Synapse Server For Private](/how-to-set-up-self-hosted-matrix-synapse-server-for-private-/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
