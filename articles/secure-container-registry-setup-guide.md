---
layout: default
title: "Secure Container Registry Setup Guide"
description: "Set up a private container registry with Harbor or registry:2, configure TLS, image scanning, signed image enforcement, and access control policies"
date: 2026-03-22
author: theluckystrike
permalink: /secure-container-registry-setup-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 6
intent-checked: true
voice-checked: true
version: '3.8'
services: registry:
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
hostname: registry.yourdomain.com
https: port: 443
  certificate: /opt/harbor/certs/fullchain.pem
  private_key: /opt/harbor/certs/privkey.pem
harbor_admin_password: "$(openssl rand -base64 24)"   # change this
database: password: "$(openssl rand -base64 24)"
trivy: ignore_unfixed: false
  skip_update: false
jobservice: max_job_workers: 10
notification: webhook_job_max_retry: 10
log: level: warning
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
  -X POST "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/latest/scan" \
  -H "Content-Type: application/json"
  "https://registry.yourdomain.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/latest/additions/vulnerabilities" \
  | jq '.["application/vnd.security.vulnerability.report; version=1.1"].vulnerabilities[] | select(.severity == "Critical")'
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  registry.yourdomain.com/myproject/myapp:v1.2.3
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
  -Bbn new-ci-user 'strong-random-password' >> /opt/registry/auth/htpasswd
  -X DELETE "https://registry.internal.example.com/v2/myapp/manifests/sha256:abcdef..."
name: Build, Push, and Scan
on: push:
    branches: [main]
jobs: build:
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
          # Harbor scans on push — poll until scan completes
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
          echo "No critical vulnerabilities — safe to deploy"
---


## Related Articles

- [Windows Registry Privacy Tweaks: A Safe Practical Guide](/windows-registry-privacy-tweaks-safe-guide/)
- [Wireguard Container Setup Docker Network Namespace Isolation](/wireguard-container-setup-docker-network-namespace-isolation/)
- [Securing Docker Containers Best Practices](/securing-docker-containers-best-practices/)
- [Container Security Basics for Developers](/container-security-basics-developers)
- [How to Set Up Authelia for SSO](/authelia-sso-setup-guide/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
