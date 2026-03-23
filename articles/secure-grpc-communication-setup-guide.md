---
layout: default
title: "Secure gRPC Communication Setup Guide"
description: "Configure gRPC with mutual TLS, server reflection controls, authentication interceptors, and rate limiting to harden service-to-service communication"
date: 2026-03-22
author: theluckystrike
permalink: /secure-grpc-communication-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Secure gRPC Communication Setup Guide

gRPC uses HTTP/2 and Protocol Buffers to provide efficient service-to-service communication. Like any RPC framework, leaving it with defaults is insecure: no TLS means plaintext traffic, no authentication means any client can call any method, and no rate limiting means a single bad actor can exhaust your service. This guide adds each security layer with working code.

## What gRPC Security Looks Like

A secured gRPC deployment has:

1. **TLS** — all traffic encrypted in transit
2. **Mutual TLS (mTLS)** — server and client both prove identity with certificates
3. **Token authentication** — calls carry a JWT or API key validated by the server
4. **Authorization interceptors** — method-level access control
5. **Reflection disabled** — service schema not exposed to unauthenticated clients
6. **Rate limiting** — per-client call rate caps

## Step 1 — Generate Certificates

Create a CA and issue certificates for server and clients:

```bash
# CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=gRPC-CA/O=MyOrg"

# Server cert
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr \
  -subj "/CN=grpc.internal/O=MyOrg"
openssl x509 -req -days 825 \
  -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

# Client cert
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr \
  -subj "/CN=grpc-client-01/O=MyOrg"
openssl x509 -req -days 825 \
  -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt
```

## Step 2 — Server with Mutual TLS (Go)

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "log"
    "net"
    "os"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    pb "myservice/proto"
)

func loadTLSCredentials() (credentials.TransportCredentials, error) {
    // Load CA cert to verify clients
    caCert, err := os.ReadFile("ca.crt")
    if err != nil {
        return nil, err
    }
    certPool := x509.NewCertPool()
    certPool.AppendCertsFromPEM(caCert)

    // Load server cert and key
    serverCert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        return nil, err
    }

    config := &tls.Config{
        Certificates: []tls.Certificate{serverCert},
        ClientAuth:   tls.RequireAndVerifyClientCert, // enforce mTLS
        ClientCAs:    certPool,
        MinVersion:   tls.VersionTLS13, // TLS 1.3 only
    }

    return credentials.NewTLS(config), nil
}

func main() {
    tlsCredentials, err := loadTLSCredentials()
    if err != nil {
        log.Fatalf("Failed to load TLS credentials: %v", err)
    }

    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }

    s := grpc.NewServer(
        grpc.Creds(tlsCredentials),
        grpc.UnaryInterceptor(authInterceptor),
    )

    pb.RegisterMyServiceServer(s, &myServiceServer{})
    fmt.Println("gRPC server listening on :50051 (mTLS)")
    s.Serve(lis)
}
```

## Step 3 — Authentication Interceptor (JWT)

Add JWT validation as a server-side interceptor that runs before every method call:

```go
import (
    "context"
    "strings"

    "github.com/golang-jwt/jwt/v5"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

var jwtSecret = []byte("your-secret-key") // use env var in production

// Methods that don't require authentication
var publicMethods = map[string]bool{
    "/myservice.MyService/HealthCheck": true,
}

func authInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Skip auth for public methods
    if publicMethods[info.FullMethod] {
        return handler(ctx, req)
    }

    // Extract metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    // Get authorization header
    authHeader := md.Get("authorization")
    if len(authHeader) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing authorization header")
    }

    // Validate Bearer token
    tokenStr := strings.TrimPrefix(authHeader[0], "Bearer ")
    token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return jwtSecret, nil
    })

    if err != nil || !token.Valid {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // Add claims to context for use in handlers
    ctx = context.WithValue(ctx, "claims", token.Claims)
    return handler(ctx, req)
}
```

## Step 4 — Client with mTLS and Token

```go
func newSecureClient() (pb.MyServiceClient, error) {
    caCert, _ := os.ReadFile("ca.crt")
    certPool := x509.NewCertPool()
    certPool.AppendCertsFromPEM(caCert)

    clientCert, _ := tls.LoadX509KeyPair("client.crt", "client.key")

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{clientCert},
        RootCAs:      certPool,
        MinVersion:   tls.VersionTLS13,
    }

    creds := credentials.NewTLS(tlsConfig)

    conn, err := grpc.Dial(
        "grpc.internal:50051",
        grpc.WithTransportCredentials(creds),
        grpc.WithUnaryInterceptor(tokenInterceptor("your-jwt-token")),
    )
    if err != nil {
        return nil, err
    }

    return pb.NewMyServiceClient(conn), nil
}

func tokenInterceptor(token string) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{},
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

## Step 5 — Disable Server Reflection in Production

gRPC server reflection lets any client with a `grpcurl` or `grpc_cli` tool enumerate all your methods. Disable it in production:

```go
// BAD — reflection enabled (development convenience)
import "google.golang.org/grpc/reflection"
reflection.Register(s) // do NOT include this in production builds

// GOOD — conditional registration
if os.Getenv("ENVIRONMENT") == "development" {
    reflection.Register(s)
}
```

Verify reflection is disabled:

```bash
# This should return an error in production
grpcurl -insecure localhost:50051 list
# Expected: Error: failed to list services: rpc error: code = Unimplemented
```

## Step 6 — Rate Limiting

Add per-client rate limiting using a token bucket interceptor:

```go
import "golang.org/x/time/rate"

type rateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.Mutex
    r        rate.Limit
    b        int
}

func newRateLimiter(r rate.Limit, b int) *rateLimiter {
    return &rateLimiter{
        limiters: make(map[string]*rate.Limiter),
        r:        r,
        b:        b,
    }
}

func (rl *rateLimiter) getLimiter(clientID string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    if l, exists := rl.limiters[clientID]; exists {
        return l
    }
    l := rate.NewLimiter(rl.r, rl.b)
    rl.limiters[clientID] = l
    return l
}

// 100 requests/second, burst of 10
var limiter = newRateLimiter(100, 10)

func rateLimitInterceptor(ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {

    md, _ := metadata.FromIncomingContext(ctx)
    clientID := "anonymous"
    if ids := md.Get("client-id"); len(ids) > 0 {
        clientID = ids[0]
    }

    if !limiter.getLimiter(clientID).Allow() {
        return nil, status.Error(codes.ResourceExhausted, "rate limit exceeded")
    }

    return handler(ctx, req)
}
```

Chain multiple interceptors:

```go
s := grpc.NewServer(
    grpc.Creds(tlsCredentials),
    grpc.ChainUnaryInterceptor(
        rateLimitInterceptor,
        authInterceptor,
        loggingInterceptor,
    ),
)
```

## Security Checklist

```
[ ] TLS 1.3 enforced (no TLS 1.0/1.1/1.2 allowed)
[ ] Mutual TLS — clients must present valid cert
[ ] JWT or API key validated on every call
[ ] Server reflection disabled in production
[ ] Rate limiting per client
[ ] Method-level authorization (not just service-level)
[ ] Certificates rotated regularly
[ ] CA private key stored offline / in HSM
```

## Related Reading

- [Secure GraphQL API Hardening Guide](/secure-graphql-api-hardening-guide/)
- [Secure MQTT Broker Setup for IoT](/secure-mqtt-broker-setup-iot/)
- [How to Set Up Cilium for Network Security](/cilium-network-security-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
