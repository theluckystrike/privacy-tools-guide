---
layout: default
title: "How to Set Up Cilium for Network Security"
description: "Deploy Cilium on Kubernetes for eBPF-based network policy enforcement, transparent encryption, and identity-aware firewall rules between services"
date: 2026-03-22
author: theluckystrike
permalink: /cilium-network-security-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up Cilium for Network Security

Cilium is a Kubernetes networking plugin that uses eBPF to enforce network policies at kernel level. Unlike iptables-based CNI plugins, Cilium operates on service identities rather than IP addresses, making policies stable across pod restarts, and enforces policies with far lower latency overhead. This guide covers installation, basic network policy enforcement, and transparent WireGuard encryption.

## Why Cilium Over Default Kubernetes Networking

Default Kubernetes networking (kube-proxy + standard CNI) provides:
- No encryption between pods
- IP-based network policies that break when pods reschedule
- No visibility into what services are actually talking to each other

Cilium provides:
- Kernel-level policy enforcement via eBPF (no iptables rules)
- Identity-based policies using Kubernetes labels
- Transparent pod-to-pod WireGuard encryption
- Layer 7 (HTTP/gRPC/Kafka) policy awareness
- Deep observability via Hubble

## Prerequisites

- Kubernetes cluster 1.25+ (kind, k3s, or managed cluster)
- Helm 3
- `kubectl` configured to reach your cluster
- Linux kernel 5.4+ on nodes

## Step 1 — Install Cilium CLI

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz

tar xzvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/

cilium version
```

## Step 2 — Install Cilium on Kubernetes

```bash
# Add Cilium Helm repo
helm repo add cilium https://helm.cilium.io/
helm repo update

# Install with WireGuard encryption and Hubble observability
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set kubeProxyReplacement=strict

# Wait for all pods to be ready
kubectl -n kube-system rollout status ds/cilium
```

Verify installation:

```bash
cilium status --wait
# Should show: Cilium:    OK (all nodes)
#              Hubble:    OK
#              Encryption: WireGuard (all nodes)
```

## Step 3 — Verify WireGuard Encryption

With `encryption.type=wireguard`, all pod-to-pod traffic crossing node boundaries is encrypted:

```bash
# Check encryption status on nodes
cilium encryption status

# Verify WireGuard tunnels are established
kubectl -n kube-system exec ds/cilium -- cilium status --verbose | grep -A5 Encryption
```

All inter-node pod traffic is now transparently encrypted. No changes needed in application code.

## Step 4 — Default Deny Network Policy

The most important first step: deny all traffic by default, then explicitly allow what's needed.

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Selects ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f default-deny.yaml
```

Now every pod in the `production` namespace is isolated. You must add explicit allow rules for each service.

## Step 5 — Allow Specific Service Communication

Example: a frontend pod needs to talk to a backend API, and the backend needs to talk to a database.

```yaml
# frontend-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egress:
  # Frontend can reach backend on port 8080
  - toEndpoints:
    - matchLabels:
        app: backend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
  # Frontend can resolve DNS
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
```

```yaml
# backend-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  # Backend accepts traffic from frontend only
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
  egress:
  # Backend can reach database on port 5432
  - toEndpoints:
    - matchLabels:
        app: postgres
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
```

```bash
kubectl apply -f frontend-policy.yaml
kubectl apply -f backend-policy.yaml
```

## Step 6 — Layer 7 HTTP Policy

Cilium can enforce policies at the HTTP level — allowing specific paths and methods only:

```yaml
# api-l7-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-http-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        # Only allow GET on /api/users
        - method: GET
          path: /api/users
        # Only allow POST on /api/orders
        - method: POST
          path: /api/orders
        # Block everything else — no rule = deny
```

```bash
kubectl apply -f api-l7-policy.yaml
```

A request to `POST /admin/delete` from the frontend would now be rejected at the kernel level, even if the application code would process it.

## Step 7 — Observe Traffic with Hubble

Hubble is Cilium's observability layer. Access it:

```bash
# Port-forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 &

# Open http://localhost:12000 in your browser
```

The Hubble UI shows a service map with live traffic flows, dropped packets, and policy verdicts.

From the command line:

```bash
# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz | \
  tar xz
sudo mv hubble /usr/local/bin/

# Port-forward Hubble relay
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &

# Watch live flows
hubble observe --follow

# Watch drops only
hubble observe --verdict DROPPED --follow

# Watch traffic from a specific pod
hubble observe --from-pod production/frontend-xxx --follow
```

## Step 8 — Block External Egress

Prevent pods from making unexpected external connections:

```yaml
# block-external-egress.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: block-external-egress
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  # Explicitly allow internal services only
  - toEndpoints:
    - matchLabels:
        app: postgres
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
  # Everything else is denied — no AWS, no Google, no external calls
```

This stops data exfiltration and supply chain attack callbacks at the network layer.

## Related Reading

- [How to Set Up Falco for Container Runtime Security](/privacy-tools-guide/falco-container-runtime-security-setup/)
- [How to Configure iptables Rules from Scratch](/privacy-tools-guide/iptables-rules-from-scratch-guide/)
- [Secure gRPC Communication Setup Guide](/privacy-tools-guide/secure-grpc-communication-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
