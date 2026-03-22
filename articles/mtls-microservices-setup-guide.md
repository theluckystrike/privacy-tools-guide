---
layout: default
title: "How to Set Up mTLS for Microservices"
description: "Configure mutual TLS authentication between microservices using a private CA, cert-manager, and Envoy or nginx to prevent unauthorized service-to-service calls"
date: 2026-03-22
author: theluckystrike
permalink: /mtls-microservices-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up mTLS for Microservices

Regular TLS authenticates the server to the client (the client verifies the server's certificate). Mutual TLS (mTLS) adds client authentication — the server also verifies the client's certificate. For microservices, this means only services with a valid certificate from your private CA can call each other, even if an attacker reaches your internal network.

## Key Takeaways

- **Topics covered**: how mtls works, step 1: set up a private certificate authority, step 2: issue service certificates
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## How mTLS Works

```
Client Service                    Server Service
     │                                   │
     │── Client certificate ──────────→  │ Verifies client cert against CA
     │← Server certificate ──────────── │
     │ Verifies server cert against CA   │
     │                                   │
     │═══════════ Encrypted ════════════ │ Both sides authenticated
```

Both the caller and the callee present certificates signed by the same private CA. If a service does not have a valid certificate, the TLS handshake fails — before any application data is exchanged.

## Step 1: Set Up a Private Certificate Authority

```bash
# Create CA private key and certificate
mkdir -p /etc/mtls-ca/{certs,private}
chmod 700 /etc/mtls-ca/private

# CA private key
openssl genrsa -out /etc/mtls-ca/private/ca.key 4096
chmod 400 /etc/mtls-ca/private/ca.key

# CA certificate (valid 10 years)
openssl req -x509 -new -nodes \
  -key /etc/mtls-ca/private/ca.key \
  -sha256 -days 3650 \
  -out /etc/mtls-ca/certs/ca.crt \
  -subj "/C=US/O=My Company/CN=Internal Services CA"

# Verify
openssl x509 -in /etc/mtls-ca/certs/ca.crt -text -noout | head -20
```

## Step 2: Issue Service Certificates

```bash
#!/bin/bash
# issue-cert.sh <service-name>
# Issues a certificate for a service, valid 90 days

SERVICE="$1"
CA_KEY="/etc/mtls-ca/private/ca.key"
CA_CERT="/etc/mtls-ca/certs/ca.crt"
OUT_DIR="/etc/mtls-certs/${SERVICE}"

mkdir -p "$OUT_DIR"

# Service private key
openssl genrsa -out "${OUT_DIR}/${SERVICE}.key" 2048

# CSR
openssl req -new \
  -key "${OUT_DIR}/${SERVICE}.key" \
  -out "${OUT_DIR}/${SERVICE}.csr" \
  -subj "/C=US/O=My Company/CN=${SERVICE}"

# Certificate config (include SANs for Kubernetes service discovery)
cat > "${OUT_DIR}/${SERVICE}.ext" << EOF
subjectAltName = DNS:${SERVICE}, DNS:${SERVICE}.default.svc.cluster.local, DNS:localhost
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
EOF

# Sign with CA
openssl x509 -req \
  -in "${OUT_DIR}/${SERVICE}.csr" \
  -CA "$CA_CERT" \
  -CAkey "$CA_KEY" \
  -CAcreateserial \
  -out "${OUT_DIR}/${SERVICE}.crt" \
  -days 90 \
  -sha256 \
  -extfile "${OUT_DIR}/${SERVICE}.ext"

echo "Issued: ${OUT_DIR}/${SERVICE}.crt"
openssl x509 -in "${OUT_DIR}/${SERVICE}.crt" -noout -dates
```

```bash
chmod +x issue-cert.sh
./issue-cert.sh user-service
./issue-cert.sh order-service
./issue-cert.sh payment-service
```

## Step 3: Configure nginx for mTLS

```nginx
# nginx.conf — server that requires client certificates
server {
    listen 443 ssl;
    server_name user-service.internal;

    # Server certificate
    ssl_certificate     /etc/mtls-certs/user-service/user-service.crt;
    ssl_certificate_key /etc/mtls-certs/user-service/user-service.key;

    # CA that signs valid client certificates
    ssl_client_certificate /etc/mtls-ca/certs/ca.crt;

    # Require client certificate — connections without a valid cert are rejected
    ssl_verify_client on;
    ssl_verify_depth 2;

    # TLS hardening
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    location / {
        # Pass the client's CN in a header for application-level logging
        proxy_set_header X-Client-CN $ssl_client_s_dn_cn;
        proxy_pass http://user-service-backend;
    }
}
```

## Step 4: Configure a Client Service (curl / Python)

```bash
# Test mTLS with curl
curl --cert /etc/mtls-certs/order-service/order-service.crt \
     --key /etc/mtls-certs/order-service/order-service.key \
     --cacert /etc/mtls-ca/certs/ca.crt \
     https://user-service.internal/api/users

# Without a valid cert — connection is rejected:
curl --cacert /etc/mtls-ca/certs/ca.crt https://user-service.internal/api/users
# curl: (56) OpenSSL SSL_read: error:14094412:SSL routines: SSL3_READ_BYTES:sslv3 alert bad certificate
```

```python
# Python requests with mTLS
import requests

response = requests.get(
    "https://user-service.internal/api/users",
    cert=(
        "/etc/mtls-certs/order-service/order-service.crt",
        "/etc/mtls-certs/order-service/order-service.key"
    ),
    verify="/etc/mtls-ca/certs/ca.crt"
)
print(response.json())
```

```go
// Go HTTP client with mTLS
package main

import (
    "crypto/tls"
    "crypto/x509"
    "net/http"
    "os"
)

func mtlsClient() (*http.Client, error) {
    cert, err := tls.LoadX509KeyPair(
        "/etc/mtls-certs/order-service/order-service.crt",
        "/etc/mtls-certs/order-service/order-service.key",
    )
    if err != nil {
        return nil, err
    }

    caCert, _ := os.ReadFile("/etc/mtls-ca/certs/ca.crt")
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
        MinVersion:   tls.VersionTLS12,
    }

    return &http.Client{
        Transport: &http.Transport{TLSClientConfig: tlsConfig},
    }, nil
}
```

## Step 5: Automate Certificate Rotation with cert-manager (Kubernetes)

In Kubernetes, cert-manager handles certificate issuance and rotation automatically.

```yaml
# issuer.yaml — ClusterIssuer using your private CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
---
# Create the secret with your CA
# kubectl create secret tls internal-ca-key-pair \
#   --cert=/etc/mtls-ca/certs/ca.crt \
#   --key=/etc/mtls-ca/private/ca.key \
#   -n cert-manager
```

```yaml
# certificate.yaml — request a cert for user-service
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: user-service-cert
  namespace: production
spec:
  secretName: user-service-tls
  duration: 2160h       # 90 days
  renewBefore: 360h     # renew 15 days before expiry
  subject:
    organizations: ["My Company"]
  dnsNames:
    - user-service
    - user-service.production.svc.cluster.local
  usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
```

```bash
kubectl apply -f issuer.yaml certificate.yaml

# cert-manager creates the secret automatically
kubectl get secret user-service-tls -n production -o yaml
```

## Verifying mTLS in Production

```bash
# Check that a service certificate is valid and signed by your CA
openssl verify -CAfile /etc/mtls-ca/certs/ca.crt /etc/mtls-certs/user-service/user-service.crt
# user-service.crt: OK

# Check certificate expiry for all services
for service in user-service order-service payment-service; do
  expiry=$(openssl x509 -in /etc/mtls-certs/${service}/${service}.crt -noout -enddate | cut -d= -f2)
  echo "${service}: expires ${expiry}"
done

# Test mTLS from within a pod
kubectl run mtls-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl --cert /path/to/client.crt --key /path/to/client.key \
  --cacert /path/to/ca.crt https://user-service/health
```

## Related Reading

- [Threat Modeling Basics for Developers](/privacy-tools-guide/threat-modeling-basics-developers/)
- [Secure Kubernetes Secrets Management Guide](/privacy-tools-guide/kubernetes-secrets-management-guide/)
- [Secure Container Registry Setup Guide](/privacy-tools-guide/secure-container-registry-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
