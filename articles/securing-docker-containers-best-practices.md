---
layout: default
title: "Securing Docker Containers Best Practices"
description: "Harden Docker deployments with non-root users, read-only filesystems, seccomp profiles, and network isolation to reduce attack surface"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /securing-docker-containers-best-practices/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, best-of]
---

{% raw %}

Docker containers share the host kernel, so a misconfigured container can expose the host to privilege escalation, data exfiltration, or full compromise. This guide covers the practical controls that meaningfully reduce attack surface in production deployments.

1. Run as a Non-Root User

The single highest-impact change: never run processes as root inside a container.

```dockerfile
FROM ubuntu:22.04

RUN groupadd -r appuser && useradd -r -g appuser -m appuser

Install dependencies as root
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip && \
    rm -rf /var/lib/apt/lists/*

Switch to non-root before the entrypoint
USER appuser
WORKDIR /home/appuser/app
COPY --chown=appuser:appuser . .

CMD ["python3", "app.py"]
```

Verify the running process:
```bash
docker exec <container_id> id
uid=999(appuser) gid=999(appuser)
```

2. Use Read-Only Filesystems

Mount the container root as read-only and explicitly allow only the directories that need writes:

```bash
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /var/run:rw,noexec,nosuid,size=16m \
  myimage:latest
```

In Docker Compose:
```yaml
services:
  app:
    image: myimage:latest
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64m
      - /var/run:noexec,nosuid,size=16m
    volumes:
      - app-data:/data:rw
```

A container with a read-only root cannot write malware to system paths or modify its own binaries.

3. Drop Linux Capabilities

Docker grants containers a large set of Linux capabilities by default. Drop all capabilities and add back only what the application needs:

```bash
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  myimage:latest
```

Common capability mappings:
| Need | Capability |
|------|-----------|
| Bind port < 1024 | `NET_BIND_SERVICE` |
| Packet capture | `NET_RAW` |
| Change file ownership | `CHOWN` |
| Manage network interfaces | `NET_ADMIN` |

Most web application containers need zero capabilities if you bind to port 8080 or above and run as a non-root user.

4. Apply a Seccomp Profile

Seccomp filters restrict which syscalls a container can make. Docker's default profile blocks ~44 dangerous syscalls. Use the strict default or create a custom profile:

```bash
Use Docker's default seccomp profile explicitly
docker run \
  --security-opt seccomp=/etc/docker/seccomp-default.json \
  myimage:latest
```

To generate a minimal profile, use `docker run` with `--security-opt seccomp=unconfined` and trace with `strace` or `sysdig`, then build an allowlist. For most apps, the default profile is sufficient.

```bash
Confirm seccomp is active
docker inspect <container_id> | grep -A5 SeccompProfile
```

5. Limit Resources

Prevent a compromised container from exhausting the host:

```bash
docker run \
  --memory="256m" \
  --memory-swap="256m" \
  --cpus="0.5" \
  --pids-limit 100 \
  myimage:latest
```

Or in Compose:
```yaml
services:
  app:
    image: myimage:latest
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    pids_limit: 100
```

`--memory-swap` equal to `--memory` disables swap for the container.

6. Network Isolation

Do not expose ports unnecessarily and use user-defined networks to isolate services:

```yaml
services:
  web:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "443:443"

  app:
    image: myapp:latest
    networks:
      - frontend
      - backend

  db:
    image: postgres:15
    networks:
      - backend
    # No ports exposed to host

networks:
  frontend:
  backend:
    internal: true  # No outbound internet access
```

The `internal - true` flag prevents containers on the `backend` network from reaching the internet, while still allowing internal service-to-service communication.

7. Prevent Privilege Escalation

Stop container processes from gaining new privileges via setuid binaries:

```bash
docker run \
  --security-opt no-new-privileges:true \
  myimage:latest
```

Add this to every production container. It is low-overhead and blocks a common escalation path.

8. Use Minimal Base Images

Fewer packages means fewer CVEs. Prefer:

- `scratch` for statically compiled binaries (Go, Rust)
- `alpine` for small footprint
- `distroless` for language runtimes without a shell

```dockerfile
Go binary with distroless
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

Distroless images contain no shell, no package manager, and no debugging tools. an attacker who achieves code execution has very limited tools to pivot.

9. Scan Images Before Deploying

```bash
Trivy. fast, complete CVE scanner
trivy image myimage:latest

Or with Docker Scout
docker scout cves myimage:latest

Fail CI if critical CVEs found
trivy image --exit-code 1 --severity CRITICAL myimage:latest
```

10. Audit the Docker Daemon

Protect the Docker socket. access to `/var/run/docker.sock` is equivalent to root on the host:

```bash
Check who can access the socket
ls -la /var/run/docker.sock
Should be - srw-rw---- 1 root docker

Never mount the socket in containers unless absolutely required
If you must, use a proxy like docker-socket-proxy with an allowlist
```

Enable Docker daemon content trust so only signed images can be pulled:

```bash
export DOCKER_CONTENT_TRUST=1
docker pull myimage:latest
Will fail if image is not signed
```

Related Articles
Image Signing and Provenance

For production systems, implement content-based security:

```bash
Sign images with Notary
docker trust signer add --key notary-keys/root_keys/root_key.key \
  mycompany-signer myregistry/myimage:latest

Verify signature before pulling
docker pull myregistry/myimage:latest
Docker verifies Notary signatures automatically with DOCKER_CONTENT_TRUST=1

For CI/CD, automatically sign images during build
docker build -t myregistry/myimage:latest .
docker trust sign myregistry/myimage:latest
docker push myregistry/myimage:latest
```

Runtime Security Monitoring

Implement continuous runtime monitoring to detect anomalies:

```bash
Use tools like Falco for runtime threat detection
sudo falco -c /etc/falco/falco.yaml

Example Falco rules for Docker containers
- rule: Suspicious Container Activity
  desc: Detect unusual syscalls
  condition: spawned_process and (
    container and (
      proc.name in (curl, wget, nc) or
      proc.args contains "/dev/shm"
    )
  )
  output: "Suspicious process in container (user=%user.name proc=%proc.name)"

Monitor for:
- Process execution from unexpected parents
- Network connections to unusual destinations
- File modifications in unexpected directories
- Privilege escalation attempts
```

Container Vulnerability Scanning in CI/CD

Integrate security scanning into your build pipeline:

```bash
#!/bin/bash
ci-pipeline-security.sh

IMAGE="myregistry/myapp:${CI_COMMIT_SHA}"

Build image
docker build -t $IMAGE .

Scan with multiple tools
echo "Running Trivy scan..."
trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE

echo "Running Grype scan..."
grype $IMAGE -f json > grype-report.json

echo "Running Snyk scan..."
snyk container test $IMAGE --exit-code=1

Only push if all scans pass
if [ $? -eq 0 ]; then
  docker push $IMAGE
  echo "Image $IMAGE passed all security scans and pushed to registry"
else
  echo "Security scans failed, image not pushed"
  exit 1
fi
```

Network Policies and Microsegmentation

For Kubernetes deployments, implement strict network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53  # DNS only
```

This policy ensures the container can only receive traffic from frontend pods and only send traffic to database pods plus DNS for domain resolution.

Secrets Management Best Practices

Never embed secrets in container images:

```bash
Bad approach - secrets in Dockerfile
ENV DATABASE_PASSWORD=secret123

Good approach - use Docker secrets
echo "dbpassword" | docker secret create db_password -

docker run --secret db_password \
  --env DATABASE_PASSWORD_FILE=/run/secrets/db_password \
  myimage:latest

Inside container:
cat /run/secrets/db_password

Or with Kubernetes:
kubectl create secret generic db-credentials \
  --from-literal=password=dbpassword

Mount as volume:
volumeMounts:
- name: db-credentials
  mountPath: /etc/secrets
  readOnly: true
```

Performance vs Security Trade-offs

Security hardening adds overhead. Profile your applications:

```bash
Benchmark read-only filesystem impact
time docker run --read-only myapp:test /bin/sh -c "heavy_workload"
time docker run myapp:test /bin/sh -c "heavy_workload"

Seccomp profile impact
time docker run --security-opt seccomp=default myapp:test workload
time docker run --security-opt seccomp=unconfined myapp:test workload

Expected overhead - 2-5% performance decrease for significant security gain
```

Related Reading

- [Wireguard Container Setup Docker Network Namespace Isolation](/wireguard-container-setup-docker-network-namespace-isolation/)
- [Nextcloud Setup Guide Raspberry Pi 2026](/nextcloud-setup-guide-raspberry-pi-2026/)
- [How to Audit Docker Images for Vulnerabilities](/how-to-audit-docker-images-for-vulnerabilities/)
- [Firefox Multi Account Containers Guide 2026](/firefox-multi-account-containers-guide-2026/)
- [Jitsi Meet Self Hosted Setup Guide](/jitsi-meet-self-hosted-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
