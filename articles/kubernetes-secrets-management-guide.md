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
score: 6
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


## Related Articles

- [1password Secrets Automation Guide](/1password-secrets-automation-guide/)
- [1password Cli Secrets Management Guide](/1password-cli-secrets-management-guide/)
- [Secure API Key Management for Developers](/secure-api-key-management-developers/)
- [1Password Secrets Automation for DevOps: A Practical Guide](/1password-secrets-automation-devops-guide/)
- [Setting Up Vault for Secrets Management](/hashicorp-vault-secrets-management-setup/)
- [AI Tools for Automated Secrets Rotation and Vault Management](https://bestremotetools.com/ai-tools-for-automated-secrets-rotation-and-vault-management/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
