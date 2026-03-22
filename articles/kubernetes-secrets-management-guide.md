---
layout: default
title: "Secure Kubernetes Secrets Management Guide"
description: "Manage Kubernetes secrets safely with encrypted etcd, Sealed Secrets, External Secrets Operator, and Vault integration to prevent secret sprawl and leaks"
date: 2026-03-22
author: theluckystrike
permalink: /kubernetes-secrets-management-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Secure Kubernetes Secrets Management Guide

Kubernetes `Secret` objects are base64 encoded, not encrypted. Anyone with read access to the cluster can decode them. Storing them in Git is even worse — they are permanently visible in history. This guide covers the full stack of Kubernetes secrets management from etcd encryption to GitOps-safe patterns.

## The Problem with Default Kubernetes Secrets

```bash
# Default secret storage — base64 is NOT encryption
kubectl create secret generic my-secret --from-literal=api-key=sk_live_abc123

# Anyone with kubectl access can trivially decode it
kubectl get secret my-secret -o jsonpath='{.data.api-key}' | base64 -d
# sk_live_abc123
```

Secrets are stored in etcd as base64. If you have etcd access (or a backup), you have the secrets.

## Step 1: Encrypt etcd at Rest

Kubernetes can encrypt etcd data using a key you provide. This protects secrets if etcd backups or snapshots are accessed.

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              # Generate: head -c 32 /dev/urandom | base64
              secret: "$(head -c 32 /dev/urandom | base64)"
      - identity: {}   # Fallback for unencrypted reads during migration
```

```bash
# Add to kube-apiserver flags (kubeadm clusters: /etc/kubernetes/manifests/kube-apiserver.yaml)
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Restart kube-apiserver (kubeadm restarts it automatically when the manifest changes)

# Verify encryption is working — existing secrets should be migrated
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
# This re-writes all existing secrets through the encryption provider

# Confirm: read directly from etcd (should see encrypted data)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key /etc/kubernetes/pki/etcd/healthcheck-client.key \
  | hexdump -C | head -4
# Should show: /registry/secrets/default/my-secret followed by encrypted binary
```

## Step 2: Sealed Secrets for GitOps

Sealed Secrets allows you to commit encrypted secrets to Git. The SealedSecret is decrypted only inside the cluster by the controller.

```bash
# Install the controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Install kubeseal CLI
curl -L https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz | tar xz
sudo mv kubeseal /usr/local/bin/
```

```bash
# Create a regular secret (do not apply this to the cluster)
kubectl create secret generic my-secret \
  --from-literal=api-key=sk_live_abc123 \
  --dry-run=client -o yaml > my-secret.yaml

# Seal it for the cluster
kubeseal --format yaml < my-secret.yaml > my-sealed-secret.yaml

# Now commit my-sealed-secret.yaml to Git — it is safe
# The content is encrypted with the cluster's public key

cat my-sealed-secret.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# spec:
#   encryptedData:
#     api-key: AgBy8ELfsd98s...  (RSA-encrypted ciphertext)
```

```bash
# Apply the sealed secret — controller decrypts it automatically
kubectl apply -f my-sealed-secret.yaml

# Verify it created a regular Secret
kubectl get secret my-secret -o jsonpath='{.data.api-key}' | base64 -d
```

### Rotating the Sealing Key

```bash
# Back up the sealing key before anything else
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml
gpg -c sealed-secrets-key-backup.yaml  # encrypt the backup

# Rotate the sealing key (creates a new key, keeps old one for decryption)
kubectl label secret -n kube-system sealed-secrets-key sealedsecrets.bitnami.com/sealed-secrets-key=active

# Re-seal all secrets with the new key
for secret in $(find . -name "*.sealed.yaml"); do
  kubeseal --re-encrypt --format yaml < "$secret" > "${secret}.new"
  mv "${secret}.new" "$secret"
done
```

## Step 3: External Secrets Operator

External Secrets Operator (ESO) pulls secrets from AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, etc., and creates Kubernetes Secrets automatically. Secrets are never stored in Git.

```bash
# Install ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

```yaml
# SecretStore — connects to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

```yaml
# ExternalSecret — defines what to pull and where to put it
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: production
spec:
  refreshInterval: 1h   # pull updates every hour
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: my-app-secret   # creates a Kubernetes Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: api-key         # key in the Kubernetes Secret
      remoteRef:
        key: prod/myapp/stripe   # AWS Secrets Manager secret name
        property: secret_key     # JSON field in the secret
    - secretKey: db-password
      remoteRef:
        key: prod/myapp/database
        property: password
```

```bash
kubectl apply -f secretstore.yaml externalSecret.yaml

# Check sync status
kubectl get externalsecret my-app-secrets -n production
# NAME               STORE                  REFRESH INTERVAL   STATUS
# my-app-secrets     aws-secretsmanager     1h                 SecretSynced
```

## Step 4: Vault Agent Sidecar

For the most granular control, Vault Agent runs as a sidecar and writes secrets to a shared memory volume.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
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
```

## Audit: What Can Access Your Secrets

```bash
# List all subjects that can read secrets in production namespace
kubectl auth can-i get secrets --namespace production --list | grep "yes"

# Or with rakkess (more detailed)
kubectl-access_matrix --namespace production

# Who can list ALL secrets cluster-wide (dangerous)
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin" or
         (.roleRef.name | test("secret"))) |
  "\(.metadata.name): \(.subjects[]?.name)"'
```

## RBAC: Minimum Necessary Access

```yaml
# Grant a service account read access to only its own secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["myapp-db", "myapp-api-keys"]  # specific secrets only
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-secret-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp
roleRef:
  kind: Role
  name: myapp-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Related Articles

- [1password Secrets Automation Guide](/privacy-tools-guide/1password-secrets-automation-guide/)
- [1password Cli Secrets Management Guide](/privacy-tools-guide/1password-cli-secrets-management-guide/)
- [Secure API Key Management for Developers](/privacy-tools-guide/secure-api-key-management-developers/)
- [1Password Secrets Automation for DevOps: A Practical Guide](/privacy-tools-guide/1password-secrets-automation-devops-guide/)
- [Setting Up Vault for Secrets Management](/privacy-tools-guide/hashicorp-vault-secrets-management-setup/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
