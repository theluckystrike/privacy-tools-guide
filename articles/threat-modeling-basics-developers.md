---
---
layout: default
title: "Threat Modeling Basics for Developers"
description: "Learn the STRIDE and PASTA frameworks for threat modeling your applications, with practical examples for web APIs, CI/CD pipelines, and data stores"
date: 2026-03-22
author: theluckystrike
permalink: /threat-modeling-basics-developers/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Threat Modeling Basics for Developers

Threat modeling is the practice of identifying what could go wrong with a system before it does. Unlike penetration testing (which finds bugs that already exist), threat modeling shapes design decisions — you build mitigations in from the start.

This guide uses STRIDE as the primary framework with practical examples you can apply to a web API, a CI/CD pipeline, and a data store.

## The STRIDE Framework

STRIDE maps threat categories to security properties:

| Threat | Breaks | Example |
|--------|--------|---------|
| **S**poofing | Authentication | Attacker impersonates a user |
| **T**ampering | Integrity | Attacker modifies data in transit |
| **R**epudiation | Non-repudiation | User denies an action they took |
| **I**nformation Disclosure | Confidentiality | API leaks user data |
| **D**enial of Service | Availability | Attacker floods the endpoint |
| **E**levation of Privilege | Authorization | Regular user accesses admin function |

## Step 1: Draw a Data Flow Diagram

A DFD shows where data comes from, where it goes, and what transforms it. You need:
- **External entities** (users, third-party services) — rectangles
- **Processes** (your code) — circles/ovals
- **Data stores** (databases, files, caches) — parallel lines
- **Data flows** (arrows showing direction)
- **Trust boundaries** (dashed lines separating zones of different trust)

```
[Browser User] ──HTTPS──> [API Gateway] ──internal──> [Auth Service]
                                │                            │
                          trust boundary                  [User DB]
                                │
                         [App Service] ──> [Postgres DB]
                                │
                         [Message Queue] ──> [Worker Service]
```

The key insight from a DFD: every arrow that crosses a trust boundary is a potential attack surface.

## Step 2: Enumerate Threats Systematically

For each element in your diagram, apply STRIDE. Use a spreadsheet or a simple text file.

```python
#!/usr/bin/env python3
# threat-model.py — generate a STRIDE threat list from your DFD components

components = [
    {"name": "API Gateway", "type": "process"},
    {"name": "Auth Service", "type": "process"},
    {"name": "User DB", "type": "datastore"},
    {"name": "HTTPS Request", "type": "dataflow", "crosses_boundary": True},
    {"name": "Internal gRPC call", "type": "dataflow", "crosses_boundary": False},
]

stride_by_type = {
    "process": ["Spoofing", "Tampering", "Repudiation", "Denial of Service", "Elevation of Privilege"],
    "datastore": ["Tampering", "Information Disclosure", "Denial of Service"],
    "dataflow": ["Tampering", "Information Disclosure", "Denial of Service"],
    "external_entity": ["Spoofing", "Repudiation"],
}

# Flows crossing trust boundaries get the full STRIDE set
for c in components:
    threats = stride_by_type.get(c["type"], [])
    if c.get("crosses_boundary"):
        threats = list(set(threats) | {"Spoofing", "Tampering", "Information Disclosure"})

    print(f"\n{c['name']} ({c['type']})")
    for t in threats:
        print(f"  [ ] {t}")
```

```bash
python3 threat-model.py > threats.txt
# Work through threats.txt, marking each as: mitigated / accepted / deferred
```

## Step 3: Rate Each Threat — DREAD or CVSS Lite

DREAD scores let you prioritize which threats to fix first.

```
DREAD scoring (1-3 each):
  Damage:          How severe is the impact?
  Reproducibility: How easy is it to reproduce?
  Exploitability:  How easy is it to exploit?
  Affected users:  How many users are affected?
  Discoverability: How easy is it to find?

Total: 5–15. 12+ = critical, 8–11 = high, 5–7 = medium, <5 = low.
```

```python
# Simple DREAD calculator
def dread_score(damage, reproducibility, exploitability, affected, discoverability):
    total = damage + reproducibility + exploitability + affected + discoverability
    if total >= 12:
        return total, "CRITICAL"
    elif total >= 8:
        return total, "HIGH"
    elif total >= 5:
        return total, "MEDIUM"
    else:
        return total, "LOW"

# Example: SQL injection in login endpoint
score, rating = dread_score(
    damage=3,           # full DB read
    reproducibility=3,  # easy, just send payload
    exploitability=2,   # needs a bit of skill
    affected=3,         # all users
    discoverability=2   # scanners find it
)
print(f"SQL injection: {score}/15 ({rating})")
# SQL injection: 13/15 (CRITICAL)
```

## Step 4: Document Mitigations

For each threat, write the control. Be specific — "use authentication" is useless, "enforce JWT validation on every route using middleware at the gateway level" is actionable.

```yaml
# threat-model.yaml — machine-readable threat register

threats:
  - id: T001
    component: API Gateway
    threat: Spoofing
    description: Attacker replays a stolen JWT to access another user's data
    risk: HIGH
    mitigation:
      control: JWT expiry set to 15 minutes; refresh tokens rotated on every use; binding JWT to client IP for admin endpoints
      status: implemented
      ticket: SEC-142

  - id: T002
    component: User DB
    threat: Information Disclosure
    description: Compromised app service reads all user records via overprivileged DB user
    risk: HIGH
    mitigation:
      control: Each service uses a dedicated DB user with SELECT on only the tables it needs; no cross-service DB access
      status: implemented
      ticket: SEC-143

  - id: T003
    component: HTTPS Request
    threat: Tampering
    description: Man-in-the-middle modifies request body before it reaches the API
    risk: MEDIUM
    mitigation:
      control: TLS 1.3 enforced; HSTS with preload; certificate pinning for mobile apps
      status: implemented
      ticket: SEC-144

  - id: T004
    component: Message Queue
    threat: Elevation of Privilege
    description: Worker consumes a crafted message that triggers admin-level actions
    risk: HIGH
    mitigation:
      control: Message schema validation before processing; no privilege granted based on message content alone
      status: in-progress
      ticket: SEC-145
```

## Applying STRIDE to a CI/CD Pipeline

CI/CD pipelines are often overlooked but have elevated privilege — they deploy to production.

```
[Developer Workstation] ──git push──> [GitHub] ──webhook──> [CI Runner]
                                                                   │
                                                          [Build + Test]
                                                                   │
                                                        [Docker Registry]
                                                                   │
                                                        [Production Deploy]
```

Key threats:

| Component | Threat | Mitigation |
|-----------|--------|------------|
| Git push | Spoofing | Require signed commits; verify author identity |
| CI Runner | Tampering | Pin action versions with SHA; read-only secrets |
| Build environment | Information Disclosure | Never log secrets; use masked variables |
| Docker Registry | Tampering | Sign images with cosign; verify before deploy |
| Deploy step | Elevation of Privilege | Least-privilege deploy token; no direct prod shell |

```bash
# Pin GitHub Actions to SHA (not tag) to prevent supply chain attacks
# Bad:  uses: actions/checkout@v4
# Good: uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

# Verify image signatures before deploy
cosign verify \
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  myregistry.io/myapp:latest
```

## Applying STRIDE to a Data Store

Databases hold your most sensitive data. Common oversights:

```sql
-- Threat: Elevation of Privilege
-- Bad: single application user with full access
GRANT ALL PRIVILEGES ON DATABASE appdb TO appuser;

-- Good: minimum required permissions per service
-- Read service: SELECT only on relevant tables
CREATE USER read_service WITH PASSWORD 'strong-random-password';
GRANT CONNECT ON DATABASE appdb TO read_service;
GRANT USAGE ON SCHEMA public TO read_service;
GRANT SELECT ON TABLE users, profiles TO read_service;

-- Write service: only on tables it must write
CREATE USER write_service WITH PASSWORD 'different-strong-password';
GRANT CONNECT ON DATABASE appdb TO write_service;
GRANT USAGE ON SCHEMA public TO write_service;
GRANT SELECT, INSERT, UPDATE ON TABLE events TO write_service;

-- No service needs DROP TABLE or TRUNCATE
```

## Quick Threat Model Template

```markdown
# Threat Model: [System Name]
Date: YYYY-MM-DD
Author: [Name]
Scope: [What is included / excluded]

## Assets (what are we protecting?)
- User PII stored in Postgres
- API authentication tokens
- Source code in private repos

## Actors
- Unauthenticated internet users
- Authenticated users
- Internal services / workers
- Developers with prod access

## Trust Boundaries
- Public internet → API Gateway (TLS termination)
- API Gateway → Internal services (mTLS)
- Services → Database (VPC-internal only)

## Threats

| ID | Component | STRIDE | Description | Risk | Mitigation | Status |
|----|-----------|--------|-------------|------|------------|--------|
| T001 | ... | ... | ... | ... | ... | ... |

## Assumptions
- VPC perimeter is not breached
- Cloud provider's managed services are trusted

## Out of Scope
- Physical attacks on data centers
- Nation-state adversaries
```

## Related Reading

- [How to Set Up Vault for Secrets Management](/privacy-tools-guide/hashicorp-vault-secrets-management-setup/)
- [How to Verify Software Supply Chain Integrity](/privacy-tools-guide/verify-software-supply-chain-integrity/)
- [How to Set Up mTLS for Microservices](/privacy-tools-guide/mtls-microservices-setup-guide/)
---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
