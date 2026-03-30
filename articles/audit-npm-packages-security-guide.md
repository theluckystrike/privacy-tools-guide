---
layout: default
title: "How to Audit npm Packages for Security"
description: "Run npm audit, use Socket and Snyk for deeper dependency analysis, detect malicious packages, and set up automated vulnerability scanning in CI"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /audit-npm-packages-security-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

How to Audit npm Packages for Security

The average Node.js project pulls in hundreds of transitive dependencies. Any one of them can introduce a vulnerability, or worse, malicious code designed to steal credentials or environment variables. This guide covers the full audit workflow from basic npm audit to CI automation.

Step 1 - npm audit

The built-in audit command checks your installed packages against the npm advisory database.

```bash
Basic audit. shows vulnerabilities and severity
npm audit

JSON output for scripting
npm audit --json | jq '.vulnerabilities | to_entries[] | {pkg: .key, severity: .value.severity, via: .value.via}'

Audit only production dependencies (skip devDependencies)
npm audit --omit=dev

Show the full dependency path for each vulnerability
npm audit --json | jq '.vulnerabilities | to_entries[] | {
  pkg: .key,
  severity: .value.severity,
  fixAvailable: .value.fixAvailable,
  nodes: .value.nodes
}'
```

Step 2 - Fix Vulnerabilities

```bash
Automatically fix vulnerabilities where a compatible update exists
npm audit fix

Force updates even if they include breaking changes (review diff first)
npm audit fix --force

Dry run. show what would change without applying
npm audit fix --dry-run

Fix a specific package manually
npm update lodash --save
```

Not all vulnerabilities can be auto-fixed. When a fix would require a major version bump, you need to evaluate manually:

```bash
Check what version is available and what changed
npm view lodash versions --json | jq '.[-5:]'
npm view lodash changelog

Update and run your test suite
npm install lodash@4.17.21
npm test
```

Step 3 - Socket.dev for Supply Chain Analysis

`npm audit` only checks known CVEs. Supply chain attacks (malicious code injected into a package) often appear before a CVE is filed. [Socket](https://socket.dev) analyzes package behavior.

```bash
Install Socket CLI
npm install -g @socketsecurity/cli

Scan your project
socket scan npm

Check a specific package before installing
socket npm info malicious-looking-package
```

Socket flags:
- Install scripts. packages that run code during `npm install`
- Network access. packages that make HTTP requests
- Filesystem access. packages that read/write files
- Obfuscated code. packages that use eval or base64 strings
- Typosquatting. packages that look like popular ones

```bash
Example output from socket scan:
lodash@4.17.21: OK
colors@1.4.0: WARNING. install script detected
faker@5.5.3: CRITICAL. obfuscated code, network access
```

Step 4 - Snyk for Vulnerability Scanning

Snyk maintains its own vulnerability database and catches issues npm audit misses.

```bash
Install and authenticate
npm install -g snyk
snyk auth   # opens browser, links to snyk.io account

Scan current project
snyk test

Monitor a project (alerts when new vulns are disclosed)
snyk monitor

Fix vulnerabilities using Snyk's patches
snyk fix

Scan a Docker image for npm vulnerabilities
snyk container test node:18-alpine
```

```bash
In CI without a Snyk account. use the free tier API
snyk test --severity-threshold=high
Exit code 1 if high or critical vulns found
```

Step 5 - Detect Malicious Package Patterns

Beyond scanners, know what to look for manually:

```bash
List all install scripts in your dependency tree
node -e "
const lock = require('./package-lock.json');
const packages = lock.packages || {};
Object.entries(packages)
  .filter(([, v]) => v.scripts && (v.scripts.install || v.scripts.preinstall || v.scripts.postinstall))
  .forEach(([name, v]) => console.log(name, v.scripts));
"

Inspect a specific package's install script
cat node_modules/suspicious-pkg/package.json | jq '.scripts'

Look for network calls in install scripts
grep -r "http\|fetch\|axios\|request\|curl\|wget" node_modules/suspicious-pkg/

Inspect for obfuscated code patterns
grep -r "eval\|atob\|Buffer.from.*base64\|String.fromCharCode" node_modules/suspicious-pkg/ | head -20
```

Step 6 - Lockfile Integrity

The `package-lock.json` file is your first line of defense. Commit it and verify it.

```bash
Install only what's in the lockfile. no surprise updates
npm ci

Never run npm install in CI (it can update the lockfile)
Always use npm ci in CI environments

Verify lockfile hasn't been tampered with
git diff HEAD package-lock.json

Check if the lockfile matches package.json
npm ls --depth=0 2>&1 | grep "UNMET\|invalid"
```

Integrity hashes are stored in `package-lock.json`:

```json
{
  "node_modules/lodash": {
    "version": "4.17.21",
    "integrity": "sha512-v2kDEe57lecTulaDIuNTPy3Ry4gLGJ6Z1O3vE1krgXZNrsQ+LFTGHVxVjcXPs17LhbZhr+33kbJlIdiwVRmA=="
  }
}
```

npm verifies this hash on every install. If a package is modified after publish, the hash will not match.

Step 7 - CI Integration

```yaml
.github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 8 * * 1'   # Weekly Monday morning scan

jobs:
  npm-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: npm audit --audit-level=high --omit=dev

      - name: Socket scan
        run: |
          npm install -g @socketsecurity/cli
          socket scan npm --strict
        continue-on-error: true   # change to false when baseline is clean

  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

Step 8 - Dependency Review for Pull Requests

GitHub's dependency review action blocks PRs that introduce vulnerable packages:

```yaml
.github/workflows/dependency-review.yml
name: Dependency Review

on:
  pull_request:

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-3.0, AGPL-3.0   # optional license policy
```

Auditing Existing Projects - Quick Script

```bash
#!/bin/bash
npm-security-check.sh. run in any Node project

echo "=== npm audit ==="
npm audit --omit=dev --json | jq '{
  total: .metadata.vulnerabilities.total,
  critical: .metadata.vulnerabilities.critical,
  high: .metadata.vulnerabilities.high
}'

echo ""
echo "=== Packages with install scripts ==="
node -e "
const lock = require('./package-lock.json');
const pkgs = lock.packages || {};
let count = 0;
Object.entries(pkgs).forEach(([name, v]) => {
  if (v.scripts && (v.scripts.install || v.scripts.postinstall || v.scripts.preinstall)) {
    console.log(' -', name);
    count++;
  }
});
console.log('Total:', count);
"

echo ""
echo "=== Dependencies older than 1 year ==="
npm outdated --json | jq -r 'to_entries[] | select(.value.current != .value.latest) | "\(.key): \(.value.current) → \(.value.latest)"' | head -20
```

Related Articles

- [VPN Provider Annual Audit Results: Independent Security](/vpn-provider-annual-audit-results-independent-security-verified/)
- [How to Audit Your Password Manager Vault: A Practical Guide](/how-to-audit-your-password-manager-vault/)
- [How to Audit Your Cloud Storage Privacy](/audit-cloud-storage-privacy-guide/)
- [How to Use Lynis for Linux Security Auditing](/lynis-linux-security-audit-guide/)
- [How to Audit Docker Images for Vulnerabilities](/how-to-audit-docker-images-for-vulnerabilities/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
