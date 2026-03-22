---
layout: default
title: "Secure Environment Variable Management"
description: "Stop leaking secrets in environment variables by using encrypted secret stores, runtime injection, and audit patterns for Docker, Kubernetes, and CI/CD"
date: 2026-03-22
author: theluckystrike
permalink: /secure-environment-variable-management-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# Secure Environment Variable Management

Storing secrets in environment variables is common and often adequate — but the default patterns are riddled with leakage vectors: `docker inspect`, process listings, crash dumps, log files, and CI/CD artifact exports all expose `ENV` values to anyone with access. This guide covers every layer from development to production.

## Why Environment Variables Are Not Enough Alone

- `docker inspect <container>` reveals all ENV values to anyone with Docker socket access
- `/proc/<pid>/environ` is readable by root and often by the process owner
- `printenv` in a shell or debug console dumps all secrets
- Crash reporters (Sentry, Rollbar) often capture environment variables
- Build systems log environment expansion: `echo $DB_PASSWORD` in a Makefile leaks to CI logs
- `.env` files accidentally committed to git (hundreds per day on GitHub)

---

## 1. Never Commit Secrets to Git

```bash
# Install git-secrets to prevent committing credentials
brew install git-secrets   # macOS
pip install detect-secrets  # cross-platform

# git-secrets: register common patterns
git secrets --install   # install hooks in current repo
git secrets --register-aws   # adds AWS key patterns
git secrets --add '[A-Z0-9]{20}'   # custom pattern

# detect-secrets: scan existing repo
detect-secrets scan > .secrets.baseline
detect-secrets audit .secrets.baseline

# Add to pre-commit hook
cat > .pre-commit-config.yaml <<'EOF'
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
EOF

pre-commit install
```

---

## 2. Use a `.env` File Correctly

For local development only:

```bash
# .env — only for local dev, NEVER commit
DB_PASSWORD=localonly_password
API_KEY=dev_key_not_real

# .gitignore
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore   # catches .env.production etc.

# Load without polluting the shell environment
# Use direnv (auto-loads .env on cd into directory)
brew install direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc

# Create .envrc (direnv reads this, not .env directly)
echo 'dotenv' > .envrc
direnv allow
```

---

## 3. Docker: Avoid ENV in Dockerfile for Secrets

```dockerfile
# BAD — secret baked into image and visible in docker inspect / history
ENV DB_PASSWORD=supersecret

# GOOD — accept at runtime, set no default
# (app reads from environment at startup)

# Use build args only for non-secret build-time values
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}
```

Pass secrets at `docker run` time from your secrets manager, not hardcoded:

```bash
# Read from a file (avoid shell history)
docker run \
  --env-file <(vault kv get -field=db_password secret/myapp | \
               awk '{print "DB_PASSWORD="$0}') \
  myapp:latest

# Or use Docker secrets (Swarm mode)
echo "supersecret" | docker secret create db_password -
docker service create \
  --secret db_password \
  --name myapp myapp:latest
# Secret available as /run/secrets/db_password — not an env var
```

Read the Docker secret in your app instead of env var:

```python
import os

def get_secret(name: str) -> str:
    # Docker secret
    secret_file = f"/run/secrets/{name}"
    if os.path.exists(secret_file):
        return open(secret_file).read().strip()
    # Fall back to env var (development)
    return os.environ[name]

DB_PASSWORD = get_secret("db_password")
```

---

## 4. Kubernetes: Use Sealed Secrets or External Secrets Operator

Native Kubernetes Secrets are base64-encoded, not encrypted. Anyone with `kubectl get secret` access can read them.

### Option A: Sealed Secrets (offline encryption)

```bash
# Install kubeseal CLI
brew install kubeseal

# Install the controller in the cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Create a sealed secret
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > sealed-db-creds.yaml

# The sealed secret is safe to commit to git
git add sealed-db-creds.yaml

# Deploy — controller decrypts and creates the real Secret
kubectl apply -f sealed-db-creds.yaml
```

### Option B: External Secrets Operator (pulls from Vault/AWS SM)

```yaml
# ExternalSecret — pulls from HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-creds
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: secret/myapp
        property: db_password
```

---

## 5. HashiCorp Vault: Dynamic Secrets

Vault goes further than static secrets — it generates short-lived, per-app credentials on demand:

```bash
# Install Vault
sudo apt install -y vault

# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp" \
  connection_url="postgresql://vault:vaultpass@localhost/mydb" \
  username="vault" password="vaultpass"

# Create a role — Vault generates credentials on request
vault write database/roles/myapp \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' \
    VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# App requests credentials at startup
vault read database/creds/myapp
# Key                Value
# username           v-myapp-aBcDeFgH
# password           A1-aBcDeFgHiJk...
# lease_duration     1h
```

The credentials expire automatically. If the app is compromised, the attacker's DB access expires within a hour.

---

## 6. Inject Secrets at Runtime (No ENV at all)

```bash
# Use envsubst + Vault agent to write config files at startup
# Or use envconsul (Consul/Vault-backed env injection)

vault agent -config=/etc/vault-agent.hcl &

# vault-agent.hcl
# template {
#   source      = "/app/config.tmpl"
#   destination = "/app/config.json"
#   command     = "/bin/kill -HUP $(cat /app/app.pid)"
# }
```

The app reads a config file written by Vault Agent — the secret is never in the environment.

---

## 7. Mask Secrets in CI/CD

GitHub Actions, GitLab CI, and CircleCI all support masked variables. Use them consistently:

```yaml
# GitHub Actions — store DB_PASSWORD in Settings > Secrets
- name: Run tests
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  run: pytest

# NEVER do this:
- run: echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> $GITHUB_ENV
# Secrets in GITHUB_ENV are visible in subsequent steps' environment dumps
```

For GitLab CI:

```yaml
# Mark variables as masked in GitLab Settings > CI/CD > Variables
# Masked values are redacted from logs automatically

test:
  variables:
    DB_PASSWORD: $DB_PASSWORD   # injected from masked variable
  script:
    - python -m pytest
```

---

## 8. Audit Running Processes for Leaked Secrets

```bash
# Check what env vars running processes expose
for pid in /proc/[0-9]*/environ; do
  echo "=== $pid ===" && tr '\0' '\n' < $pid 2>/dev/null | grep -iE "pass|key|secret|token"
done

# Scan Docker containers
for cid in $(docker ps -q); do
  echo "=== $cid ===" && docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' $cid \
    | grep -iE "pass|key|secret|token"
done
```

---

## Checklist

- [ ] `.env` in `.gitignore` and pre-commit hook prevents commit
- [ ] No `ENV` secret instructions in Dockerfiles
- [ ] Docker secrets or Vault used for production containers
- [ ] Kubernetes Secrets encrypted at rest (etcd encryption) or replaced by ESO
- [ ] CI/CD variables marked as masked/protected
- [ ] Vault dynamic secrets used for database credentials
- [ ] Crash reporter env capture disabled in production

---

## Related Reading

- [Secure JWT Implementation Best Practices](/privacy-tools-guide/secure-jwt-implementation-best-practices/)
- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [Secure Database Connection Pooling Guide](/privacy-tools-guide/secure-database-connection-pooling-guide/)
- [Secure Kubernetes Secrets Management Guide](/privacy-tools-guide/kubernetes-secrets-management-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
