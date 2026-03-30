---
layout: default
title: "Secure Container Registry Setup Guide"
description: "Set up a private container registry with Harbor or registry:2, configure TLS, image scanning, signed image enforcement, and access control policies"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /secure-container-registry-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Secure Container Registry Setup Guide

Running your own container registry keeps images off Docker Hub (where they are public by default), eliminates vendor lock-in, and lets you enforce image scanning and signature verification before anything gets deployed. This guide covers both the lightweight `registry:2` option and the more full-featured Harbor.

Option 1 - Docker Registry (registry:2)

`registry:2` is the official, minimal registry. It stores images and handles auth. It does not include a vulnerability scanner or web UI.

```bash
Create directories
mkdir -p /opt/registry/{data,certs,auth}

Generate TLS certificate
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout /opt/registry/certs/domain.key \
  -x509 -days 3650 \
  -out /opt/registry/certs/domain.crt \
  -subj "/CN=registry.internal.example.com"

Generate htpasswd credentials
docker run --rm --entrypoint htpasswd httpd:alpine \
  -Bbn alice 'strong-password-here' > /opt/registry/auth/htpasswd

docker run --rm --entrypoint htpasswd httpd:alpine \
  -Bbn ci-bot 'ci-bot-password' >> /opt/registry/auth/htpasswd
```

```yaml
/opt/registry/docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    restart: always
    ports:
      - "443:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_LOG_LEVEL: warn
    volumes:
      - /opt/registry/data:/var/lib/registry
      - /opt/registry/certs:/certs
      - /opt/registry/auth:/auth
```

```bash
cd /opt/registry && docker compose up -d

Test push and pull
docker login registry.internal.example.com -u alice
docker pull nginx:alpine
docker tag nginx:alpine registry.internal.example.com/nginx:alpine
docker push registry.internal.example.com/nginx:alpine
docker pull registry.internal.example.com/nginx:alpine
```

Option 2 - Harbor (Full-Featured)

Harbor adds - role-based access control, vulnerability scanning with Trivy, image signing, audit logs, and a web UI. Recommended for teams.

```bash
Download Harbor installer
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-online-installer-v2.11.0.tgz
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-online-installer-v2.11.0.tgz.asc

Verify GPG signature
gpg --keyserver keyserver.ubuntu.com --recv-keys 644FF454C0B4115C
gpg --verify harbor-online-installer-v2.11.0.tgz.asc harbor-online-installer-v2.11.0.tgz

tar xzf harbor-online-installer-v2.11.0.tgz
cd harbor
```

```yaml
harbor.yml (key settings)
hostname: registry.yourdomain.com

https:
  port: 443
  certificate: /opt/harbor/certs/fullchain.pem
  private_key: /opt/harbor/certs/privkey.pem

harbor_admin_password: "$(openssl rand -base64 24)"   # change this

database:
  password: "$(openssl rand -base64 24)"

trivy:
  ignore_unfixed: false
  skip_update: false

jobservice:
  max_job_workers: 10

notification:
  webhook_job_max_retry: 10

log:
  level: warning
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
```

```bash
sudo ./install.sh --with-trivy
```

Enabling Vulnerability Scanning (Harbor)

Harbor integrates Trivy for automatic image scanning on push.

In the Harbor web UI:
1. Administration → Interrogation Services → Vulnerability → Configure
2. Enable auto-scan: scan new images on push
3. Set scan schedule: daily at 2am

Or via API:

```bash
Trigger a scan on a specific image
curl -u admin:password \
  -X POST "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/latest/scan" \
  -H "Content-Type: application/json"

Get scan report
curl -u admin:password \
  "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/latest/additions/vulnerabilities" \
  | jq '.["application/vnd.security.vulnerability.report; version=1.1"].vulnerabilities[] | select(.severity == "Critical")'
```

Enforcing Signed Images Only

cosign signing (Sigstore)

```bash
Install cosign
curl -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign && sudo mv cosign /usr/local/bin/

Sign an image after pushing
cosign sign --yes registry.yourdomain.com/myproject/myapp:v1.2.3

Verify before deploy
cosign verify \
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  registry.yourdomain.com/myproject/myapp:v1.2.3
```

Harbor Content Trust Policy

In Harbor web UI - Project → Configuration → Enable content trust → Prevent vulnerable images

Set a severity threshold:
- Images with Critical vulnerabilities: block
- High: warn
- Medium and below: allow

Access Control and User Management

```bash
Harbor - create a robot account for CI with push access to specific project
curl -u admin:password \
  -X POST "https://registry.yourdomain.com/api/v2.0/projects/myproject/robots" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ci-bot",
    "duration": 90,
    "permissions": [{
      "kind": "project",
      "namespace": "myproject",
      "access": [
        {"resource": "repository", "action": "push"},
        {"resource": "repository", "action": "pull"},
        {"resource": "artifact", "action": "read"}
      ]
    }]
  }' | jq '{name: .name, token: .secret}'

The token is only shown once. store it in your CI secrets
```

```bash
registry:2: manage users with htpasswd
Add a new CI user
docker run --rm --entrypoint htpasswd httpd:alpine \
  -Bbn new-ci-user 'strong-random-password' >> /opt/registry/auth/htpasswd

Remove a user
sed -i '/^old-user:/d' /opt/registry/auth/htpasswd

Registry does not require restart. htpasswd changes take effect immediately
```

Image Cleanup

Registries accumulate stale images. Automate cleanup:

```bash
registry:2: garbage collection (removes unreferenced layers)
Stop writes first (or use --dry-run to check)
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run

Harbor - retention policies via web UI or API
Project → Tag Immutability (prevent overwriting signed tags)
Project → Tag Retention → Add rule: keep last 10 versions per repo

Manual delete via registry API
curl -u alice:password \
  -X DELETE "https://registry.internal.example.com/v2/myapp/manifests/sha256:abcdef..."
```

Security Checklist

```bash
Verify registry TLS is not accepting weak protocols
nmap --script ssl-enum-ciphers -p 443 registry.yourdomain.com

Check for exposed registry with no auth
curl -s https://registry.yourdomain.com/v2/ | jq .
Should return - {"errors":[{"code":"UNAUTHORIZED",...}]}
Never - {} (open with no auth)

Scan the registry image itself for vulnerabilities
trivy image registry:2
trivy image goharbor/harbor-core:v2.11.0
```

Integrating Harbor with CI/CD Pipelines

Configure your CI pipeline to push images to Harbor after every successful build and enforce scanning before deployment:

```yaml
.github/workflows/build-and-push.yml
name: Build, Push, and Scan
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Harbor
        uses: docker/login-action@v3
        with:
          registry: registry.yourdomain.com
          username: ${{ secrets.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: registry.yourdomain.com/myproject/myapp:${{ github.sha }}

      - name: Wait for Harbor vulnerability scan
        run: |
          # Harbor scans on push. poll until scan completes
          MAX_RETRIES=20
          for i in $(seq 1 $MAX_RETRIES); do
            STATUS=$(curl -s -u "${{ secrets.HARBOR_USERNAME }}:${{ secrets.HARBOR_PASSWORD }}" \
              "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/${{ github.sha }}" \
              | jq -r '.scan_overview."application/vnd.security.vulnerability.report; version=1.1".scan_status // "not_started"')
            echo "Scan status: $STATUS (attempt $i/$MAX_RETRIES)"
            [ "$STATUS" = "Success" ] && break
            [ "$i" = "$MAX_RETRIES" ] && echo "Scan timed out" && exit 1
            sleep 15
          done

      - name: Check for critical vulnerabilities
        run: |
          CRITICAL=$(curl -s -u "${{ secrets.HARBOR_USERNAME }}:${{ secrets.HARBOR_PASSWORD }}" \
            "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/${{ github.sha }}/additions/vulnerabilities" \
            | jq '[.["application/vnd.security.vulnerability.report; version=1.1"].vulnerabilities[] | select(.severity == "Critical")] | length')

          echo "Critical vulnerabilities: $CRITICAL"
          [ "$CRITICAL" -gt 0 ] && echo "Build failed: critical CVEs found" && exit 1
          echo "No critical vulnerabilities. safe to deploy"
```

---

Related Reading

- [How to Verify Software Supply Chain Integrity](/verify-software-supply-chain-integrity/)
- [How to Set Up mTLS for Microservices](/mtls-microservices-setup-guide/)
- [Secure Kubernetes Secrets Management Guide](/kubernetes-secrets-management-guide/)
- [Secure WebSocket Connections Setup Guide](/secure-websocket-connections-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [AI Container Security Scanning](https://bestremotetools.com/ai-container-security-scanning/)

---

Related Articles

- [Windows Registry Privacy Tweaks: A Safe Practical Guide](/windows-registry-privacy-tweaks-safe-guide/)
- [Wireguard Container Setup Docker Network Namespace Isolation](/wireguard-container-setup-docker-network-namespace-isolation/)
- [Securing Docker Containers Best Practices](/securing-docker-containers-best-practices/)
- [Container Security Basics for Developers](/container-security-basics-developers)
- [How to Set Up Authelia for SSO](/authelia-sso-setup-guide/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
