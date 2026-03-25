---
layout: default
title: "How to Set Up GnuTLS for Application Security"
description: "Integrate GnuTLS into C and Python applications to add TLS 1.3, certificate pinning, mutual TLS, and secure cipher configuration to your network code"
date: 2026-03-22
author: theluckystrike
permalink: /gnutls-application-security-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}
How to Set Up GnuTLS for Application Security

GnuTLS is the TLS library preferred on GNU/Linux systems (used by GLib, curl, libsoup, and GNOME applications). It has a different API philosophy from OpenSSL. credential-based rather than context-based. and excellent support for TLS 1.3, DTLS, and mutual certificate authentication. This guide covers practical integration in C and Python, certificate pinning, and system-level hardening.

Install GnuTLS

```bash
Ubuntu / Debian
sudo apt install -y libgnutls28-dev gnutls-bin gnutls-doc python3-gnutls

RHEL / Fedora
sudo dnf install -y gnutls gnutls-devel gnutls-utils

Verify version (3.8+ recommended for TLS 1.3 support)
gnutls-cli --version
```

---

Generate Certificates with certtool

GnuTLS ships `certtool`, which is more scriptable than OpenSSL's CA workflow:

```bash
Generate a private key (Ed25519 for modern security)
certtool --generate-privkey --key-type ed25519 --outfile server-key.pem

Generate a self-signed certificate for testing
certtool --generate-self-signed --load-privkey server-key.pem \
  --outfile server-cert.pem <<'EOF'
Country name - US
State - CA
Locality - San Francisco
Organization - Example Corp
Common name - server.example.com
Does the certificate belong to an authority? no
Is this a TLS web server certificate? yes
Enter the dnsName of the subject of the certificate: server.example.com
Is the above information ok? yes
EOF

Generate CA + signed server cert (production pattern)
certtool --generate-privkey --key-type rsa --bits 4096 --outfile ca-key.pem
certtool --generate-self-signed --load-privkey ca-key.pem \
  --template ca.cfg --outfile ca-cert.pem

ca.cfg template:
cat > ca.cfg <<'EOF'
cn = "Example CA"
ca
cert_signing_key
expiration_days = 3650
EOF

certtool --generate-privkey --key-type ed25519 --outfile app-key.pem
certtool --generate-request --load-privkey app-key.pem \
  --template server.cfg --outfile app-req.pem

certtool --generate-certificate \
  --load-request app-req.pem \
  --load-ca-certificate ca-cert.pem \
  --load-ca-privkey ca-key.pem \
  --template server.cfg --outfile app-cert.pem
```

---

Command-Line TLS Testing

```bash
Connect to a server and show certificate details
gnutls-cli --print-cert server.example.com

Test with specific TLS version
gnutls-cli --priority "NORMAL:-VERS-TLS-ALL:+VERS-TLS1.3" server.example.com

Test with mutual TLS (client presents a certificate)
gnutls-cli --x509certfile client-cert.pem --x509keyfile client-key.pem \
  server.example.com

Verify a certificate against a CA bundle
gnutls-cli --x509cafile ca-cert.pem --no-system-cas server.example.com

Check supported cipher suites for a server
gnutls-cli --list server.example.com
```

---

C Integration - Minimal TLS Client

```c
/* tls_client.c. GnuTLS 3.x minimal client */
#include <gnutls/gnutls.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>

#define SERVER   "server.example.com"
#define PORT     "443"
#define CAFILE   "/etc/ssl/certs/ca-certificates.crt"

int tcp_connect(const char *host, const char *port) {
    struct addrinfo hints = {0}, *res;
    hints.ai_socktype = SOCK_STREAM;
    getaddrinfo(host, port, &hints, &res);
    int fd = socket(res->ai_family, res->ai_socktype, 0);
    connect(fd, res->ai_addr, res->ai_addrlen);
    freeaddrinfo(res);
    return fd;
}

int main(void) {
    gnutls_global_init();

    gnutls_certificate_credentials_t creds;
    gnutls_certificate_allocate_credentials(&creds);

    /* Load system CA bundle */
    gnutls_certificate_set_x509_trust_file(creds, CAFILE, GNUTLS_X509_FMT_PEM);

    /* Set TLS 1.3 only with strong ciphers */
    gnutls_session_t session;
    gnutls_init(&session, GNUTLS_CLIENT);
    gnutls_priority_set_direct(session,
        "SECURE256:+VERS-TLS1.3:-VERS-TLS-ALL", NULL);

    gnutls_credentials_set(session, GNUTLS_CRD_CERTIFICATE, creds);
    gnutls_server_name_set(session, GNUTLS_NAME_DNS, SERVER, strlen(SERVER));

    /* Connect and hand off to GnuTLS */
    int fd = tcp_connect(SERVER, PORT);
    gnutls_transport_set_int(session, fd);
    gnutls_handshake_set_timeout(session, GNUTLS_DEFAULT_HANDSHAKE_TIMEOUT);

    int ret;
    do { ret = gnutls_handshake(session); }
    while (ret < 0 && gnutls_error_is_fatal(ret) == 0);

    if (ret < 0) {
        fprintf(stderr, "Handshake failed: %s\n", gnutls_strerror(ret));
        goto cleanup;
    }

    /* Verify certificate */
    unsigned int status;
    gnutls_certificate_verify_peers3(session, SERVER, &status);
    if (status != 0) {
        gnutls_datum_t out;
        gnutls_certificate_verification_status_print(
            status, GNUTLS_CRT_X509, &out, 0);
        fprintf(stderr, "Certificate verification failed: %s\n", out.data);
        gnutls_free(out.data);
        goto cleanup;
    }

    printf("TLS version: %s\n", gnutls_protocol_get_name(
           gnutls_protocol_get_version(session)));
    printf("Cipher: %s\n", gnutls_cipher_get_name(
           gnutls_cipher_get(session)));

    /* Send HTTP GET */
    const char *req = "GET / HTTP/1.1\r\nHost: " SERVER "\r\nConnection: close\r\n\r\n";
    gnutls_record_send(session, req, strlen(req));

    char buf[4096];
    while ((ret = gnutls_record_recv(session, buf, sizeof(buf)-1)) > 0) {
        buf[ret] = '\0';
        printf("%s", buf);
    }

cleanup:
    gnutls_bye(session, GNUTLS_SHUT_RDWR);
    close(fd);
    gnutls_deinit(session);
    gnutls_certificate_free_credentials(creds);
    gnutls_global_deinit();
    return 0;
}
```

```bash
gcc -o tls_client tls_client.c $(pkg-config --cflags --libs gnutls)
./tls_client
```

---

Python Integration with python-gnutls

```python
from gnutls.library.functions import *
from gnutls.crypto import *
import socket

Simpler approach - use Python's ssl module (backed by GnuTLS on Debian/Ubuntu)
import ssl

def create_gnutls_context(cafile=None, certfile=None, keyfile=None,
                          pin=None):
    """
    Create an SSL context using GnuTLS as the backend.
    pin: SHA-256 hex fingerprint of the expected server certificate (optional).
    """
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.minimum_version = ssl.TLSVersion.TLSv1_3
    ctx.verify_mode = ssl.CERT_REQUIRED
    ctx.check_hostname = True

    # Strong cipher list (GnuTLS priority string via OpenSSL compat)
    ctx.set_ciphers("TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256")

    if cafile:
        ctx.load_verify_locations(cafile)
    else:
        ctx.load_default_certs()

    if certfile and keyfile:
        ctx.load_cert_chain(certfile, keyfile)

    return ctx, pin

def connect_with_pin(host, port, ctx, expected_pin=None):
    with socket.create_connection((host, port)) as raw_sock:
        with ctx.wrap_socket(raw_sock, server_hostname=host) as tls_sock:
            if expected_pin:
                import hashlib
                cert_der = tls_sock.getpeercert(binary_form=True)
                actual_pin = hashlib.sha256(cert_der).hexdigest()
                if actual_pin != expected_pin:
                    raise ssl.SSLCertVerificationError(
                        f"Certificate pin mismatch: expected {expected_pin}, "
                        f"got {actual_pin}"
                    )
            return tls_sock.version(), tls_sock.cipher()

Usage
ctx, pin = create_gnutls_context(
    pin="a3b4c5d6e7..."   # SHA-256 of expected server cert DER
)
version, cipher = connect_with_pin("server.example.com", 443, ctx, pin)
print(f"Connected: {version} / {cipher[0]}")
```

---

Priority Strings Reference

GnuTLS uses priority strings to configure allowed algorithms:

```bash
TLS 1.3 only, strong ciphers
"SECURE256:+VERS-TLS1.3:-VERS-TLS-ALL"

All modern versions (1.2 + 1.3), no weak ciphers
"NORMAL:-3DES-CBC:-RC4-128"

mTLS: require client certificate
"NORMAL:%REQUIRE_CRT_IN_HANDSHAKE"

FIPS-compatible
"FIPS140-2"

Test what a priority string enables
gnutls-cli --list --priority "SECURE256"
```

---

Setting System-Wide GnuTLS Policy

On Fedora/RHEL with system crypto policies:

```bash
View current policy
update-crypto-policies --show

Upgrade to FUTURE policy (TLS 1.2+, RSA 3072+)
sudo update-crypto-policies --set FUTURE

Or set custom policy
sudo tee /etc/crypto-policies/policies/custom.pol > /dev/null <<'EOF'
min_tls_version = TLS1.2
min_dtls_version = DTLS1.2
key_size = RSA >= 3072
hash = SHA2-256+
EOF

sudo update-crypto-policies --set CUSTOM
```

---

Related Reading

- [Secure JWT Implementation Best Practices](/secure-jwt-implementation-best-practices/)
- [How to Set Up ModSecurity WAF with Nginx](/modsecurity-waf-nginx-setup-guide/)
- [Secure API Gateway Setup with Kong](/kong-api-gateway-secure-setup-guide/)
- [How to Set Up age Encryption for Teams](/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
