---
layout: default
title: "Container Security Basics for Developers"
description: "Practical container security for developers. Covers image hardening, non-root users, secrets management, network policies, and runtime security."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: container-security-basics-developers
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

Most container security issues come from a handful of avoidable mistakes: running as root, using bloated base images, baking secrets into layers, and leaving default network access open. This guide covers the practical steps developers can take before a security team ever gets involved.

## The Basic Threat Model

A container is not a VM. The kernel is shared with the host. If a container process escapes (via a kernel vulnerability or misconfiguration), it can interact with the host or other containers. The goal is to minimize what a compromised container can do.

Key principles:
- Least privilege: containers should have only what they need to run
- Minimal attack surface: small images, no unnecessary tools
- Immutable: containers should not write to their own filesystem
- No secrets in images: credentials must be injected at runtime

## Use Minimal Base Images

Large base images include package managers, shells, and utilities that attackers can use if they get in. Prefer minimal alternatives:

| Instead of | Use |
|---|---|
| `ubuntu:latest` | `ubuntu:minimal` or `debian:slim` |
| `python:3.12` | `python:3.12-slim` or `python:3.12-alpine` |
| `node:20` | `node:20-slim` or `node:20-alpine` |
| Any general base | `gcr.io/distroless/base` |

Distroless images contain only the runtime and your app — no shell, no package manager, nothing for an attacker to use.

Example Dockerfile using distroless for a Go binary:

```dockerfile
# Build stage
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -o server ./cmd/server

# Final stage — no shell, no package manager
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

The final image contains only the static binary and the distroless base. `docker scout` or `trivy` will find very few CVEs to report on it.

## Never Run as Root

By default, container processes run as root (UID 0). If your app is exploited, the attacker has root inside the container and a better chance of escaping.

Add a dedicated user in your Dockerfile:

```dockerfile
FROM node:20-slim

WORKDIR /app

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1001 appuser

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN chown -R appuser:appgroup /app

# Switch to non-root
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

Verify a running container is not root:

```bash
docker exec my-container id
# Should show uid=1001(appuser) gid=1001(appgroup)
```

For Kubernetes, enforce this in the pod security context:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    fsGroup: 1001
  containers:
  - name: myapp
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

## Never Bake Secrets into Images

Docker images are layered. Even if you delete a file in a later layer, the original layer still contains the secret. Anyone who pulls the image can extract it.

**Wrong:**

```dockerfile
ENV DB_PASSWORD=supersecret123
RUN echo "password=supersecret123" > /app/config.ini
```

**Right — inject secrets at runtime:**

```bash
# Pass via environment variable
docker run -e DB_PASSWORD="$DB_PASSWORD" myapp

# Or use Docker secrets (Swarm)
docker secret create db_password ./db_password.txt
docker service create --secret db_password myapp

# Or mount a secrets file
docker run -v /run/secrets:/run/secrets:ro myapp
```

In Kubernetes, use Secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: <base64-encoded-password>
---
spec:
  containers:
  - name: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

For production, use a proper secrets manager (HashiCorp Vault, AWS Secrets Manager) rather than Kubernetes Secrets, which are only base64-encoded by default.

## Scan Images for Vulnerabilities

Build scanning into your CI pipeline:

```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan an image
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan and fail CI if critical vulns found
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan your Dockerfile for misconfigurations
trivy config --severity HIGH,CRITICAL Dockerfile
```

Integrate into a GitHub Actions workflow:

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: 1
```

## Restrict Container Capabilities

Linux capabilities break root privileges into granular permissions. Drop all capabilities and add back only what's needed:

```bash
# Drop all capabilities, add only NET_BIND_SERVICE (for port < 1024)
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Or in docker-compose.yml
services:
  myapp:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

Common capabilities you almost never need:

- `SYS_ADMIN` — effectively root, avoid unless required
- `NET_ADMIN` — network configuration
- `SYS_PTRACE` — process tracing
- `DAC_OVERRIDE` — bypass file permission checks

## Read-Only Filesystem

Make the container filesystem read-only and mount writable volumes only where needed:

```bash
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  -v /data/app-uploads:/app/uploads \
  myapp
```

In docker-compose:

```yaml
services:
  myapp:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    volumes:
      - app-uploads:/app/uploads
```

A read-only root filesystem prevents attackers from writing backdoors, modifying binaries, or installing tools.

## Network Policies

By default, all containers on a Docker network can talk to each other. Use explicit network segmentation:

```yaml
services:
  web:
    networks:
      - frontend
      - backend

  api:
    networks:
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true   # No internet access
```

In Kubernetes, use NetworkPolicies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432
```

## Runtime Security Monitoring

Falco is an open source runtime security tool that detects anomalous behavior in containers:

```bash
# Install Falco
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt update && sudo apt install falco
```

Falco detects events like:
- Shell spawned inside a container
- Sensitive file read (`/etc/shadow`, `/proc/*/mem`)
- Package manager executed inside a container
- Network connection from unexpected container

## Quick Security Checklist

- [ ] Non-root user configured in Dockerfile
- [ ] Minimal base image (slim, alpine, or distroless)
- [ ] No secrets in Dockerfile or image layers
- [ ] Image scanned with Trivy in CI
- [ ] Capabilities dropped (`--cap-drop=ALL`)
- [ ] Read-only root filesystem
- [ ] Network segmentation between services
- [ ] Resource limits set (CPU and memory)
- [ ] Health checks defined
- [ ] Container logs shipped to centralized logging



## Related Reading

- [Best Hardware Security Key for Developers: A Practical Guide](/privacy-tools-guide/best-hardware-security-key-for-developers/)
- [Proton Pass vs Bitwarden Security Comparison for Developers](/privacy-tools-guide/proton-pass-vs-bitwarden-security-comparison/)
- [Tor Browser Isolation Container Setup Guide](/privacy-tools-guide/tor-browser-isolation-container-setup-guide/)
- [Wireguard Container Setup Docker Network Namespace Isolation](/privacy-tools-guide/wireguard-container-setup-docker-network-namespace-isolation/)
- [Age Encryption Tool Tutorial for Developers](/privacy-tools-guide/age-encryption-tool-tutorial-developers/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
