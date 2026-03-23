---
layout: default
title: "How to Audit Docker Images for Vulnerabilities"
description: "Scan Docker images for CVEs using Trivy, Grype, and Docker Scout, then integrate scanning into CI/CD pipelines to block vulnerable deployments"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-audit-docker-images-for-vulnerabilities/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Shipping a Docker image without scanning it is like deploying code without testing — you are pushing unknown vulnerabilities to production. Most containers are built on base images with hundreds of packages, each with its own CVE history. This guide covers scanning images locally, integrating into CI/CD, and interpreting results.

## Tools Overview

| Tool | Database | Best For |
|------|----------|----------|
| Trivy | GHSA, NVD, OS advisories | Fast,, CI integration |
| Grype | Anchore's vulnerability DB | SBOM integration |
| Docker Scout | Docker's advisory DB | Native Docker tooling |

## Trivy: The Fast Standard

### Install Trivy

```bash
# Ubuntu/Debian
sudo apt install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install trivy

# macOS
brew install trivy
```

### Scan an Image

```bash
# Scan a local image
trivy image nginx:1.25

# Only show HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL nginx:1.25

# Exit with error code 1 if any CRITICAL found (for CI)
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# JSON output
trivy image --format json --output scan-results.json myapp:latest

# Include secrets and misconfigurations
trivy image --scanners vuln,secret,misconfig myapp:latest

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed myapp:latest
```

### Scan Dockerfile Before Building

```bash
trivy config Dockerfile
trivy config .
```

## Grype: SBOM-First Scanning

```bash
# Install
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | \
  sudo sh -s -- -b /usr/local/bin

# Scan
grype nginx:1.25

# Generate SBOM then scan
syft nginx:1.25 -o cyclonedx-json > nginx-sbom.json
grype sbom:nginx-sbom.json

# Fail on critical
grype --fail-on critical nginx:1.25
```

## Docker Scout

```bash
docker scout cves nginx:1.25
docker scout compare nginx:1.24 nginx:1.25
docker scout recommendations nginx:1.25
docker scout quickview myapp:latest
```

## CI/CD Integration

### GitHub Actions with Trivy

```yaml
# .github/workflows/scan.yml
name: Container Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

### GitLab CI

```yaml
container-scan:
  image: aquasec/trivy:latest
  stage: test
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Trivy Ignore Files

When a CVE has no fix or is a false positive:

```bash
# .trivyignore
CVE-2024-12345
CVE-2024-99999 exp:2026-09-01
```

## Interpreting Results

Prioritize by:
1. CVSS score 9.0+: patch immediately
2. Network-reachable component: higher priority
3. Fix available: upgrade immediately if a patched version exists
4. EPSS score: probability of exploitation in the wild

```bash
# Extract CVSS scores for criticals
trivy image --format json myapp:latest | \
  jq '[.Results[].Vulnerabilities[] |
       select(.Severity == "CRITICAL") |
       {vuln: .VulnerabilityID, cvss: .CVSS.nvd.V3Score}]'
```

## SBOM Generation: Software Bill of Materials

An SBOM documents every package in your image. This is required by many enterprises and government contracts.

```bash
# Generate SBOM with Syft (Anchore's tool)
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | \
  sudo sh -s -- -b /usr/local/bin

# Generate CycloneDX format (most common)
syft myapp:latest -o cyclonedx-json > myapp-sbom.json

# Generate SPDX format (alternative standard)
syft myapp:latest -o spdx-json > myapp-sbom-spdx.json

# View the SBOM
jq '.components | length' myapp-sbom.json
# Example output: 847 (847 packages in the image)
```

An SBOM is essential for:
- Audit trails (regulatory compliance)
- Vulnerability tracking (when a new CVE appears, you know immediately if you're affected)
- Supply chain security (detecting compromised dependencies)
- License compliance (tracking open-source licenses)

## Private Registry Scanning

If using a private Docker registry, scan images immediately after push:

```bash
# Configure registry credentials
trivy image --registry-username user --registry-password token \
  my-private-registry.com/myapp:latest

# Or use Docker config
trivy image --skip-update my-private-registry.com/myapp:latest
```

Combine with webhooks to automate:

```bash
# Setup: Your registry sends webhook to a scanning endpoint
# Scanning script: scan-webhook.sh

#!/bin/bash
IMAGE_NAME=$1
IMAGE_TAG=$2

trivy image --exit-code 1 \
  --severity CRITICAL,HIGH \
  "${IMAGE_NAME}:${IMAGE_TAG}"

if [ $? -ne 0 ]; then
    # Send alert
    curl -X POST https://slack.com/api/chat.postMessage \
      -H "Authorization: Bearer $SLACK_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"channel\": \"#security\", \"text\": \"Image $IMAGE_NAME:$IMAGE_TAG has critical vulnerabilities\"}"
    exit 1
fi

# Tag and push
docker tag "${IMAGE_NAME}:${IMAGE_TAG}" \
  my-registry.com/scanned/"${IMAGE_NAME}:${IMAGE_TAG}"
docker push my-registry.com/scanned/"${IMAGE_NAME}:${IMAGE_TAG}"
```

## Enforcing Minimal Base Images

Use the smallest base images to reduce the attack surface:

| Base Image | Size | Packages | CVEs (typical) |
|-----------|------|----------|----------------|
| ubuntu:latest | 77 MB | 350+ | 20-50 |
| debian:bookworm-slim | 65 MB | 200+ | 15-30 |
| alpine:latest | 7 MB | 50+ | 2-5 |
| scratch (empty) | 0 MB | 0 | 0 |

```dockerfile
# Bad: Based on full Ubuntu
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl wget git
# ... 450 MB final image

# Better: Slim variant
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y curl
# ... 150 MB final image

# Best: Multi-stage with alpine
FROM golang:1.21-alpine as builder
WORKDIR /src
COPY . .
RUN go build -o myapp

FROM alpine:latest
COPY --from=builder /src/myapp /usr/local/bin/
# ... 50 MB final image

# Optimal: Scratch for compiled languages
FROM scratch
COPY --from=builder /src/myapp /myapp
# ... 15 MB final image (just the binary)
```

For interpreted languages (Python, Node, Java), scratch is not possible, but alpine reduces size by 90%.

## Scanning Dockerfiles Before Build

Scan the Dockerfile itself for misconfigurations:

```bash
trivy config Dockerfile

# Example output:
# HIGH: Missing USER instruction (runs as root)
# MEDIUM: Using latest tag instead of pinned version
# LOW: No healthcheck defined
```

Common misconfigurations to avoid:

```dockerfile
# BAD: Runs as root
FROM alpine:latest
RUN apk add curl

# GOOD: Creates non-root user
FROM alpine:latest
RUN apk add curl && \
    addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser

# BAD: Uses latest tag
FROM ubuntu:latest

# GOOD: Uses pinned version with digest
FROM ubuntu:22.04@sha256:abcdef0123456789

# BAD: Single layer, large image
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y build-essential && \
    apt-get install -y nodejs && \
    apt-get install -y git && \
    apt-get clean

# GOOD: Multi-stage build
FROM ubuntu:22.04 as builder
RUN apt-get update && apt-get install -y build-essential

FROM alpine:latest
COPY --from=builder /app /app
```

## Continuous Scanning: Registry Monitoring

For production registries, scan all images weekly:

```bash
#!/bin/bash
# scan-all-images.sh

REGISTRY="my-registry.com"
REPO="production"

# List all images
curl -s https://${REGISTRY}/v2/${REPO}/_catalog | \
  jq -r '.repositories[]' | while read IMAGE; do

  # List all tags for this image
  curl -s https://${REGISTRY}/v2/${IMAGE}/tags/list | \
    jq -r '.tags[]' | while read TAG; do

    echo "Scanning ${IMAGE}:${TAG}"
    trivy image --severity HIGH,CRITICAL \
      "${REGISTRY}/${IMAGE}:${TAG}" \
      --exit-code 0  # Don't fail, just report
  done
done
```

Run via cron:
```bash
# /etc/cron.weekly/docker-scan
0 2 * * 0 root /path/to/scan-all-images.sh 2>&1 | \
  mail -s "Weekly Docker Scan Report" security@example.com
```

## Signing Docker Images

Prevent unauthorized image tampering with signatures:

```bash
# Install Notary (Docker's signing tool)
# Download from: https://github.com/notaryproject/notary/releases

# Create signing key (you'll be prompted for a passphrase)
notary key generate --rootkey

# Sign an image
notary addhash my-registry.com/myapp latest <image-digest>

# Verify signature before running
docker pull my-registry.com/myapp:latest
notary verify my-registry.com/myapp
```

Push signed images and verify in CI/CD:

```yaml
# .github/workflows/verify-image.yml
- name: Verify image signature
  run: |
    notary verify my-registry.com/myapp:latest
    docker run my-registry.com/myapp:latest
```

Images without valid signatures are rejected during deployment.

## Related Articles

- [Nextcloud Setup Guide Raspberry Pi 2026](/nextcloud-setup-guide-raspberry-pi-2026/)
- [Securing Docker Containers Best Practices](/securing-docker-containers-best-practices/)
- [Jitsi Meet Self Hosted Setup Guide](/jitsi-meet-self-hosted-setup-guide/)
- [How to Audit npm Packages for Security](/audit-npm-packages-security-guide/)
- [How to Set Up a Tor Relay](/how-to-set-up-tor-relay-node/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
