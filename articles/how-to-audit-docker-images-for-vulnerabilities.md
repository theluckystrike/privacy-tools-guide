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
| Trivy | GHSA, NVD, OS advisories | Fast, comprehensive, CI integration |
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

## Related Articles

- [Nextcloud Setup Guide Raspberry Pi 2026](/privacy-tools-guide/nextcloud-setup-guide-raspberry-pi-2026/)
- [Securing Docker Containers Best Practices](/privacy-tools-guide/securing-docker-containers-best-practices/)
- [Jitsi Meet Self Hosted Setup Guide](/privacy-tools-guide/jitsi-meet-self-hosted-setup-guide/)
- [How to Audit npm Packages for Security](/privacy-tools-guide/audit-npm-packages-security-guide/)
- [How to Set Up a Tor Relay](/privacy-tools-guide/how-to-set-up-tor-relay-node/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
