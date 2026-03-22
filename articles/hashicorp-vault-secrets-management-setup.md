---
layout: default
title: "Setting Up Vault for Secrets Management"
description: "Deploy HashiCorp Vault, configure secret engines, enable dynamic credentials, set up policies, and integrate Vault with Kubernetes and CI/CD pipelines"
date: 2026-03-22
author: theluckystrike
permalink: /hashicorp-vault-secrets-management-setup/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Setting Up Vault for Secrets Management

HashiCorp Vault centralizes secret storage, enforces access policies, and issues short-lived dynamic credentials. Instead of hardcoding a database password in your app, Vault issues a temporary credential that expires in 1 hour. When it's gone, so is the attack surface.

## Key Takeaways

- **You need 3 of**: the 5 to unseal.
- **Topics covered**: installation, development mode (testing only), production installation with systemd
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Installation

```bash
# Debian/Ubuntu
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault

# RHEL/CentOS
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault

# Verify
vault version
```

## Development Mode (Testing Only)

```bash
# Start Vault in dev mode (in-memory, pre-unsealed, auto-root-token)
vault server -dev -dev-root-token-id="root"

# In another terminal:
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault status
vault secrets list
```

Dev mode stores nothing permanently — data is lost on restart. Never use in production.

## Production Installation with systemd

```hcl
# /etc/vault.d/vault.hcl
ui = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault.crt"
  tls_key_file  = "/opt/vault/tls/vault.key"
}

api_addr     = "https://vault.internal.example.com:8200"
cluster_addr = "https://vault.internal.example.com:8201"
```

```bash
# Create data directory and permissions
sudo mkdir -p /opt/vault/data /opt/vault/tls
sudo chown -R vault:vault /opt/vault

# Generate TLS cert (or use Let's Encrypt)
openssl req -x509 -newkey rsa:4096 -days 3650 -nodes \
  -keyout /opt/vault/tls/vault.key \
  -out /opt/vault/tls/vault.crt \
  -subj "/CN=vault.internal.example.com"

sudo systemctl enable --now vault

# Initialize Vault (first time only)
vault operator init -key-shares=5 -key-threshold=3
```

`init` outputs 5 unseal keys and 1 root token. Store each key in a different location (different person, different password manager). You need 3 of the 5 to unseal.

```bash
# Unseal (requires 3 keys)
vault operator unseal  # paste key 1
vault operator unseal  # paste key 2
vault operator unseal  # paste key 3

vault status  # Sealed: false
```

## KV Secrets Engine

```bash
export VAULT_ADDR='https://vault.internal.example.com:8200'
export VAULT_TOKEN='hvs.your-root-token'

# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/database \
  username="appuser" \
  password="$(openssl rand -base64 24)"

# Read a secret
vault kv get secret/myapp/database

# Read as JSON
vault kv get -format=json secret/myapp/database | jq '.data.data'

# Update a single field (KV v2 patches)
vault kv patch secret/myapp/database password="new-password"

# List versions
vault kv metadata get secret/myapp/database

# Roll back to a previous version
vault kv rollback -version=2 secret/myapp/database
```

## Dynamic Database Credentials

This is Vault's most powerful feature for databases. Vault creates a temporary database user, hands the credentials to your app, and deletes the user when the TTL expires.

```bash
# Enable the database secrets engine
vault secrets enable database

# Configure a PostgreSQL connection
vault write database/config/my-postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.internal:5432/appdb" \
  username="vault_admin" \
  password="$(cat /etc/vault/pg-admin-password)"
```

```sql
-- On the PostgreSQL server, create Vault's admin user with permission to create roles
CREATE ROLE vault_admin WITH LOGIN PASSWORD 'strong-password' CREATEROLE;
GRANT CONNECT ON DATABASE appdb TO vault_admin;
```

```bash
# Create a role that defines what the temporary credentials can do
vault write database/roles/app-role \
  db_name=my-postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Request credentials
vault read database/creds/app-role
# Key                Value
# ---                -----
# lease_duration     1h
# username           v-token-app-role-AbCdEfGhIj
# password           A1B2C3D4E5F6...
```

Your application requests credentials at startup. When it crashes or restarts, it gets new credentials — the old ones are already deleted.

## Policies

Policies define what a token can access.

```hcl
# app-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "database/creds/app-role" {
  capabilities = ["read"]
}

# Deny everything else
path "*" {
  capabilities = ["deny"]
}
```

```bash
vault policy write app-policy app-policy.hcl

# Create a token tied to this policy
vault token create -policy="app-policy" -ttl=24h

# Verify the policy
vault token lookup <token>
```

## Kubernetes Integration (Vault Agent)

Vault Agent runs as a sidecar, authenticates to Vault, and writes secrets to files for your application.

```yaml
# kubernetes-auth setup
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert="$(kubectl config view --raw --minify -o json | jq -r '.clusters[0].cluster["certificate-authority-data"]' | base64 -d)"

# Create a role binding the service account to the policy
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=production \
  policies=app-policy \
  ttl=1h
```

```yaml
# pod-with-vault-agent.yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: myapp
  initContainers:
    - name: vault-agent
      image: hashicorp/vault:latest
      command: ["vault", "agent", "-config=/vault/config/agent.hcl"]
      volumeMounts:
        - name: vault-config
          mountPath: /vault/config
        - name: secrets
          mountPath: /vault/secrets
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: secrets
          mountPath: /var/secrets
          readOnly: true
  volumes:
    - name: vault-config
      configMap:
        name: vault-agent-config
    - name: secrets
      emptyDir:
        medium: Memory  # tmpfs — secrets never written to disk
```

## Related Reading

- [Secure API Key Management for Developers](/privacy-tools-guide/secure-api-key-management-developers/)
- [Secure Kubernetes Secrets Management Guide](/privacy-tools-guide/kubernetes-secrets-management-guide/)
- [Threat Modeling Basics for Developers](/privacy-tools-guide/threat-modeling-basics-developers/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
