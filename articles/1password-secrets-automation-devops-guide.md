---
layout: default
title: "1Password Secrets Automation for DevOps: A Practical Guide"
description: "Learn how to integrate 1Password secrets automation into your DevOps workflows. Covers Terraform, Ansible, GitLab CI, Kubernetes, and production-ready"
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /1password-secrets-automation-devops-guide/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, automation]
---


{% raw %}

To automate secrets with 1Password in DevOps, create a dedicated vault, set up a service account with scoped permissions, then use the 1Password CLI (`op`) or Connect server to inject credentials into Terraform, Ansible, Kubernetes, or your CI/CD pipeline at runtime. This eliminates hardcoded credentials from repos and environment files while providing audit trails and access controls. Below are production-ready examples for each integration pattern.

## Setting Up 1Password for DevOps Automation

Before integrating secrets into your workflows, establish a dedicated vault for automation and create service accounts with appropriate permissions. Service accounts differ from user accounts—they provide API access without requiring human authentication, making them ideal for CI/CD runners and infrastructure scripts.

Create a new vault specifically for DevOps secrets:

```bash
# Sign in and create a dedicated automation vault
op vault create "Infrastructure Secrets"
```

Next, create a service account through the 1Password admin console with access to this vault. Service accounts support granular permissions—grant only the specific vault access needed for each automation use case. Note the service account token upon creation; you'll use this in environment variables and secrets management systems.

## Integrating 1Password with Terraform

Infrastructure as Code requires secure handling of API keys, database credentials, and cloud service tokens. The 1Password provider for Terraform enables dynamic secret retrieval during infrastructure provisioning.

First, configure the 1Password provider in your Terraform configuration:

```hcl
terraform {
  required_providers {
    onepassword = {
      source = "1password/onepassword"
      version = "1.0.0"
    }
  }
}

provider "onepassword" {
  url = "https://mycompany.1password.com"
  token = var.onepassword_token
}

resource "onepassword_item" "aws_production" {
  vault = "Infrastructure Secrets"
  item {
    title = "AWS Production Keys"
    category = "API Credential"
    fields {
      name = "access_key_id"
      value = var.aws_access_key_id
      type = "STRING"
    }
    fields {
      name = "secret_access_key"
      value = var.aws_secret_access_key
      type = "CONCEALED"
    }
  }
}
```

For simpler use cases, retrieve secrets directly using the CLI and inject them as environment variables:

```bash
#!/bin/bash
# terraform-secrets.sh

export AWS_ACCESS_KEY_ID=$(op item get "AWS Production" --field "access_key_id")
export AWS_SECRET_ACCESS_KEY=$(op item get "AWS Production" --field "secret_access_key")

terraform init
terraform apply -auto-approve
```

This approach keeps sensitive values out of your Terraform state file while maintaining reproducible infrastructure deployments.

## Ansible Integration Patterns

Ansible playbooks often require credentials for cloud providers, database connections, and API tokens. Rather than embedding secrets in vars files or Ansible Vault, retrieve them dynamically from 1Password.

Create a lookup plugin wrapper for 1Password in your Ansible setup:

```bash
# Install 1Password CLI on Ansible control node
brew install --cask 1password-cli
```

Use the shell command lookup in playbooks:

```yaml
---
- name: Configure production database
  hosts: db_servers
  vars:
    db_password: "{{ lookup('command', 'op item get Database --field password') }}"
  tasks:
    - name: Update database password
      community.postgresql.postgresql_user:
        name: app_user
        password: "{{ db_password }}"
        db: application
        login_host: "{{ inventory_hostname }}"
      no_log: true
```

For production Ansible deployments, use the service account token approach with environment variables:

```bash
export OP_SERVICE_ACCOUNT_TOKEN="your_token_here"
ansible-playbook -i inventory/production site.yml
```

## Kubernetes Secret Management

Kubernetes secrets require careful handling to avoid exposing credentials in etcd or container images. Several approaches exist for integrating 1Password with Kubernetes.

The simplest method generates Kubernetes secrets directly from 1Password:

```bash
#!/bin/bash
# generate-k8s-secrets.sh

kubectl create secret generic app-secrets \
  --from-literal=db-password=$(op item get "Production Database" --field password) \
  --from-literal=redis-password=$(op item get "Redis Cache" --field password) \
  --from-literal=api-key=$(op item get "External API" --field key) \
  --dry-run=client -o yaml > k8s/secrets.yaml
```

For more sophisticated implementations, the External Secrets Operator supports 1Password as a provider. Install the operator and configure it to sync secrets automatically:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword-store
spec:
  provider:
    onepassword:
      connectHost: "https://mycompany.1password.io"
      auth:
        secretRef:
          opServiceAccountToken:
            name: onepassword-credentials
            namespace: external-secrets
            key: token
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: onepassword-store
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: "Production Database"
        property: password
```

This approach automatically synchronizes secrets from 1Password into Kubernetes, refreshing them periodically without manual intervention.

## GitLab CI/CD Implementation

GitLab CI/CD integrates smoothly with 1Password using the service account token:

```yaml
stages:
  - build
  - deploy

variables:
  OP_SERVICE_ACCOUNT_TOKEN: $OP_SERVICE_ACCOUNT_TOKEN

before_script:
  - |
    if ! command -v op &> /dev/null; then
      curl -sSf https://app-updates.agilebits.com/product_history/CLI2 | \
      grep -E "op_.*\.pkg" | head -1 | \
      awk -F'{print $4}' | \
      xargs -I {} brew install {}
    fi
    echo "$OP_SERVICE_ACCOUNT_TOKEN" | op signin --stdin

build:
  stage: build
  script:
    - echo "DB_PASSWORD=$(op item get ProductionDB --field password)" >> .env
    - echo "API_KEY=$(op item get ExternalAPI --field key)" >> .env
    - docker build --build-arg DB_PASSWORD="$(op item get ProductionDB --field password)" .
  artifacts:
    paths:
      - .env
    expire_in: 1 hour

deploy:
  stage: deploy
  script:
    - |
      export DEPLOY_KEY="$(op item get DeployKey --field private_key)"
      echo "$DEPLOY_KEY" > deploy_key
      chmod 600 deploy_key
      ssh -i deploy_key deploy@production.example.com "./deploy.sh"
```

This configuration installs the 1Password CLI in the job, authenticates using the service account token, retrieves secrets as needed, and uses them within each job without persisting them to logs or artifacts.

## Production Considerations

When implementing 1Password automation at scale, several practices improve security and reliability.

Session management matters in long-running processes. The CLI caches credentials temporarily—for extended automation workflows, periodically refresh the session:

```bash
# Check session validity
op account get || op signin --stdin < /dev/null
```

Network security becomes critical when transmitting credentials between systems. Always use TLS and avoid logging secret values:

```bash
# Wrong - exposes password in process list
export PASSWORD=$(op item get "Item" --field password)
echo $PASSWORD # Never do this

# Correct - suppress command output
export PASSWORD=$(op item get "Item" --field password 2>/dev/null)
```

Access auditing provides visibility into which automation systems accessed which secrets. Review the 1Password audit logs regularly to identify unusual access patterns or unauthorized attempts.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)
- [1Password CLI Secrets Management Guide: A Practical.](/privacy-tools-guide/1password-cli-secrets-management-guide/)
- [1Password Secrets Automation Guide: Integrating.](/privacy-tools-guide/1password-secrets-automation-guide/)
- [How to Audit Your Password Manager Vault: A Practical Guide](/privacy-tools-guide/how-to-audit-your-password-manager-vault/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
