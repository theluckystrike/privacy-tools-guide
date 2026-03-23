---
layout: default
title: "How to Implement mTLS Between Microservices"
description: "Configure mutual TLS between services using cert-manager, Vault PKI, and Istio to enforce service identity, prevent lateral movement, and automate"
date: 2026-03-22
author: theluckystrike
permalink: /secure-microservice-communication-mtls-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
How to Implement mTLS Between Microservices

Mutual TLS (mTLS) is the most reliable way to enforce identity between microservices. Both the client service and the server service present X.509 certificates; connections from services without a valid certificate are rejected at the TLS layer. before any application code runs. This guide implements mTLS at three levels: manual (for understanding), cert-manager (for Kubernetes), and Vault PKI (for multi-cloud).

Why mTLS Over API Keys

API keys can be stolen, logged, and shared. They don't expire automatically and don't identify which specific instance of a service is making a call. An X.509 certificate with a 24-hour TTL issued to a specific pod identity is far harder to abuse.

```
API Key threat model:
  Stolen key → attacker has permanent access until manual rotation

mTLS threat model:
  Stolen certificate → attacker has access until cert expires (24h typical)
  Certificate is tied to a specific service identity (SPIFFE ID or CN)
  Cannot be used from a different IP without a reissued cert
```

---

Part 1: Manual mTLS Setup (Foundation)

Create a Private CA

```bash
Generate CA private key and certificate
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -out ca.crt -days 3650 \
  -subj "/O=MyOrg/CN=Internal Services CA"

Set secure permissions
chmod 600 ca.key
```

Issue Service Certificates

```bash
issue_cert() {
  local SERVICE="$1"
  # Private key
  openssl genrsa -out "${SERVICE}.key" 2048

  # CSR with SAN for service DNS name
  openssl req -new -key "${SERVICE}.key" -out "${SERVICE}.csr" \
    -subj "/O=MyOrg/CN=${SERVICE}" \
    -addext "subjectAltName=DNS:${SERVICE},DNS:${SERVICE}.default.svc.cluster.local"

  # Sign with CA. short 24h TTL forces rotation
  openssl x509 -req -in "${SERVICE}.csr" -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out "${SERVICE}.crt" -days 1 \
    -extensions v3_req \
    -extfile <(printf "[v3_req]\nsubjectAltName=DNS:%s,DNS:%s.default.svc.cluster.local" \
                      "$SERVICE" "$SERVICE")
  rm "${SERVICE}.csr"
}

issue_cert "auth-service"
issue_cert "payment-service"
issue_cert "order-service"
```

Test mTLS Connection

```bash
Server (payment-service)
openssl s_server -accept 8443 \
  -cert payment-service.crt -key payment-service.key \
  -CAfile ca.crt \
  -Verify 1 \   # require client certificate
  -tls1_3 &

Client (auth-service connecting to payment-service)
openssl s_client -connect localhost:8443 \
  -cert auth-service.crt -key auth-service.key \
  -CAfile ca.crt \
  -tls1_3

Connection rejected without client cert:
openssl s_client -connect localhost:8443 -CAfile ca.crt
SSL_ERROR_HANDSHAKE_FAILURE_ALERT
```

---

Part 2: cert-manager on Kubernetes

cert-manager automates certificate issuance, renewal, and distribution as Kubernetes Secrets.

Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

Wait for pods
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager --timeout=120s
```

Create an Internal CA Issuer

```yaml
internal-ca.yaml. create CA secret and issuer
---
apiVersion: v1
kind: Secret
metadata:
  name: internal-ca-key-pair
  namespace: cert-manager
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-ca.crt>
  tls.key: <base64-encoded-ca.key>
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

```bash
Encode CA cert and key
kubectl create secret tls internal-ca-key-pair \
  --cert=ca.crt --key=ca.key -n cert-manager

kubectl apply -f internal-ca.yaml
```

Request Certificates for Each Service

```yaml
auth-service-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: auth-service-tls
  namespace: default
spec:
  secretName: auth-service-tls
  duration: 24h           # 24-hour TTL
  renewBefore: 1h         # Renew 1h before expiry
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  dnsNames:
    - auth-service
    - auth-service.default.svc.cluster.local
  usages:
    - client auth
    - server auth
```

```bash
kubectl apply -f auth-service-cert.yaml
kubectl apply -f payment-service-cert.yaml

Verify cert was issued
kubectl get certificate auth-service-tls
NAME               READY   SECRET             AGE
auth-service-tls   True    auth-service-tls   30s
```

Mount Certificates in Pods

```yaml
Deployment snippet for auth-service
spec:
  template:
    spec:
      volumes:
        - name: tls-certs
          secret:
            secretName: auth-service-tls
        - name: ca-cert
          secret:
            secretName: internal-ca-key-pair
            items:
              - key: tls.crt
                path: ca.crt
      containers:
        - name: auth-service
          image: myorg/auth-service:latest
          volumeMounts:
            - name: tls-certs
              mountPath: /certs
              readOnly: true
            - name: ca-cert
              mountPath: /certs/ca
              readOnly: true
          env:
            - name: TLS_CERT_FILE
              value: /certs/tls.crt
            - name: TLS_KEY_FILE
              value: /certs/tls.key
            - name: TLS_CA_FILE
              value: /certs/ca/ca.crt
```

---

Part 3: Vault PKI Secrets Engine

For multi-cloud or non-Kubernetes environments, HashiCorp Vault's PKI engine issues short-lived certificates on demand.

```bash
Enable PKI secrets engine
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki   # 10-year CA

Generate internal CA
vault write pki/root/generate/internal \
  common_name="Internal Services CA" \
  ttl=87600h

Create intermediate CA for services
vault secrets enable -path=pki_int pki
vault write pki_int/intermediate/generate/internal \
  common_name="Services Intermediate CA"
  # Get CSR, sign with root, import back

Create role for service certs
vault write pki_int/roles/services \
  allowed_domains="svc.cluster.local,internal" \
  allow_subdomains=true \
  max_ttl=24h \
  require_cn=false \
  server_flag=true \
  client_flag=true
```

Issue a cert for a service (called at container startup via init container):

```bash
In an init container or entrypoint script
vault write pki_int/issue/services \
  common_name="payment-service.default.svc.cluster.local" \
  alt_names="payment-service" \
  ttl=24h \
  -format=json | python3 -c "
import json, sys, os
data = json.load(sys.stdin)['data']
open('/certs/service.crt', 'w').write(data['certificate'])
open('/certs/service.key', 'w').write(data['private_key'])
open('/certs/ca.crt', 'w').write(data['issuing_ca'])
os.chmod('/certs/service.key', 0o600)
print('Certificate issued, expires:', data['expiration'])
"
```

---

Part 4: Verify mTLS in Production

```bash
Check that a service rejects connections without client cert
kubectl exec -it debug-pod -- \
  curl -v --cacert /certs/ca.crt \
  https://payment-service.default.svc.cluster.local:8443/health
Should fail: "SSL peer certificate or SSH remote key was not OK"

Verify with client cert
kubectl exec -it debug-pod -- \
  curl -v \
  --cacert /certs/ca.crt \
  --cert /certs/tls.crt \
  --key /certs/tls.key \
  https://payment-service.default.svc.cluster.local:8443/health
Should succeed: HTTP 200

Check certificate TTL
echo | openssl s_client -connect payment-service:8443 \
  -CAfile /certs/ca.crt \
  -cert /certs/tls.crt \
  -key /certs/tls.key 2>/dev/null \
  | openssl x509 -noout -dates
```

---

Automating Certificate Rotation

```bash
If using cert-manager: automatic (watches Secret, triggers renewal)
If using Vault: use Vault Agent with auto-renew

Vault Agent config for automatic renewal
cat > /etc/vault-agent.hcl <<'EOF'
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = { role = "payment-service" }
  }
}

template {
  source      = "/etc/tls.tmpl"
  destination = "/certs/service.crt"
  command     = "systemctl reload payment-service"
  perms       = 0644
}
EOF

Template file for certificate
{{- with secret "pki_int/issue/services" "common_name=payment-service" "ttl=24h" -}}
{{ .Data.certificate }}
{{- end }}
```

---

Related Reading

- [Secure Microservice Communication Patterns](/secure-microservice-communication-patterns/)
- [Secure JWT Implementation Best Practices](/secure-jwt-implementation-best-practices/)
- [Secure Environment Variable Management](/secure-environment-variable-management-guide/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
