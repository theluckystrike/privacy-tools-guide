---
layout: default
title: "Secure Kubernetes Secrets Management Guide"
description: "Manage Kubernetes secrets safely with encrypted etcd, Sealed Secrets, External Secrets Operator, and Vault integration to prevent secret sprawl and leaks"
date: 2026-03-22
author: theluckystrike
permalink: /kubernetes-secrets-management-guide/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
resources: - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              # Generate: head -c 32 /dev/urandom | base64
              secret: "$(head -c 32 /dev/urandom | base64)"
      - identity: {}   # Fallback for unencrypted reads during migration
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key /etc/kubernetes/pki/etcd/healthcheck-client.key \
  | hexdump -C | head -4
  --from-literal=api-key=sk_live_abc123 \
  --dry-run=client -o yaml > my-secret.yaml
  kubeseal --re-encrypt --format yaml < "$secret" > "${secret}.new"
  mv "${secret}.new" "$secret"
metadata: name: myapp-secret-reader
  namespace: production
spec: template:
    spec:
      serviceAccountName: myapp
      # Vault Agent init container — fetches secrets before app starts
      initContainers:
        - name: vault-agent-init
          image: hashicorp/vault:latest
          command: ["vault", "agent", "-config=/vault/config/agent.hcl", "-exit-after-auth"]
          volumeMounts:
            - name: vault-config
              mountPath: /vault/config
            - name: secrets
              mountPath: /vault/secrets
      containers:
        - name: app
          image: myapp:latest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-db   # created by Vault Agent template
                  key: password
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
            medium: Memory   # never written to disk
  .items[] |
  select(.roleRef.name == "cluster-admin" or
         (.roleRef.name | test("secret"))) |
  "\(.metadata.name): \(.subjects[]?.name)"'
rules: - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["myapp-db", "myapp-api-keys"]  # specific secrets only
    verbs: ["get"]
---


## Preventing Secrets in Git History

Even with Sealed Secrets or ESO, developers sometimes accidentally commit raw credentials. Add a pre-commit hook that scans for high-entropy strings and known secret patterns:

```bash
# Install detect-secrets
pip install detect-secrets

# Initialize a baseline (commits current state as known)
detect-secrets scan > .secrets.baseline
git add .secrets.baseline

# Add pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
detect-secrets-hook --baseline .secrets.baseline
EOF
chmod +x .git/hooks/pre-commit
```

For team-wide enforcement use pre-commit framework:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

If a secret has already been committed, assume it is compromised and rotate it immediately before removing it from history:

```bash
# Remove a file from all history (requires force push — coordinate with team)
git filter-repo --path secrets.yaml --invert-paths

# Or use BFG (faster for large repos)
bfg --delete-files secrets.yaml
git reflog expire --expire=now --all && git gc --prune=now
git push --force
```

Removing from history does not help if the secret was already exposed — rotate it first, then clean history for hygiene.

---

## Related Articles

- [1password Secrets Automation Guide](/1password-secrets-automation-guide/)
- [1password Cli Secrets Management Guide](/1password-cli-secrets-management-guide/)
- [Secure API Key Management for Developers](/secure-api-key-management-developers/)
- [1Password Secrets Automation for DevOps: A Practical Guide](/1password-secrets-automation-devops-guide/)
- [Setting Up Vault for Secrets Management](/hashicorp-vault-secrets-management-setup/)
- [AI Tools for Automated Secrets Rotation and Vault Management](https://bestremotetools.com/ai-tools-for-automated-secrets-rotation-and-vault-management/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
