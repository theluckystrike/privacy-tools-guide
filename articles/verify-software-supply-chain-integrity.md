---
layout: default
title: "How to Verify Software Supply Chain Integrity"
description: "Verify downloads with checksums and signatures, use cosign for container images, verify GitHub Actions with pinned SHAs, and check SBOM for your builds"
date: 2026-03-22
author: theluckystrike
permalink: /verify-software-supply-chain-integrity/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

How to Verify Software Supply Chain Integrity

Supply chain attacks inject malicious code between the developer and you. in the build system, the package registry, the release artifacts, or the CI pipeline. Verification cannot stop every attack, but it confirms the software you are running matches what the author intended to ship.

Verifying Downloads with Checksums

Most projects publish SHA256 checksums alongside their releases.

```bash
Download a release and its checksum file
wget https://releases.example.com/myapp-2.1.0-linux-amd64.tar.gz
wget https://releases.example.com/myapp-2.1.0-SHA256SUMS

Verify
sha256sum -c myapp-2.1.0-SHA256SUMS
myapp-2.1.0-linux-amd64.tar.gz: OK

If you only have the expected hash (no checksum file)
echo "a3f2b1c4d5e6... myapp-2.1.0-linux-amd64.tar.gz" | sha256sum -c
```

Checksums prove the file was not corrupted in transit, but they do not prove the file came from the developer. the checksum file could be replaced alongside the binary. GPG or Sigstore signatures provide stronger guarantees.

Verifying GPG Signatures

```bash
Import the developer's public key
gpg --keyserver keys.openpgp.org --recv-keys 0xABCDEF1234567890

Or from a project's own keyserver URL
curl https://example.com/signing-key.asc | gpg --import

Verify the signature
gpg --verify myapp-2.1.0-linux-amd64.tar.gz.asc myapp-2.1.0-linux-amd64.tar.gz
Good signature from "Developer Name <dev@example.com>"

Check the key fingerprint matches what the project documents
gpg --fingerprint 0xABCDEF1234567890
```

A good signature means:
- The file was signed by the owner of that key
- The file has not been modified since signing

It does not mean the key itself was not compromised (that requires a trust chain or key transparency).

Sigstore / cosign for Container Images

Sigstore is the modern replacement for GPG-signed container images. GitHub Actions can sign images automatically during CI using keyless signing (OIDC-based).

```bash
Install cosign
curl -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign
sudo mv cosign /usr/local/bin/

Verify a signed image
cosign verify \
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:v2.1.0

Output - Verified OK
Shows - signature, certificate, build workflow reference
```

```yaml
In your GitHub Actions workflow. sign the image during build
- name: Sign the container image
  env:
    DIGEST: ${{ steps.build.outputs.digest }}
  run: |
    cosign sign --yes ghcr.io/myorg/myapp@${DIGEST}
```

The keyless model ties the signature to the GitHub Actions workflow identity. you can verify not just "who signed" but "which pipeline signed, from which repo, on which branch."

Pinning GitHub Actions to Commit SHAs

Referencing an action by tag (`actions/checkout@v4`) means the tag can be moved to point to different code. Pin to a SHA instead.

```bash
Find the SHA for a tag
gh api repos/actions/checkout/git/refs/tags/v4 --jq '.object.sha'
Returns a tag SHA, then dereference to the commit:
gh api repos/actions/checkout/git/tags/abc123... --jq '.object.sha'

Or check the commit directly on GitHub
github.com/actions/checkout/commits/v4
```

```yaml
Bad. tag can be updated
uses: actions/checkout@v4

Good. pinned to commit SHA
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

Use [pin-github-actions](https://github.com/mheap/pin-github-actions) to automate this:

```bash
npx pin-github-actions .github/workflows/*.yml
```

SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts) defines four levels of build integrity guarantees.

| Level | Requirement |
|-------|-------------|
| SLSA 1 | Provenance generated (who built it, when, from what source) |
| SLSA 2 | Signed provenance |
| SLSA 3 | Provenance from a hardened build platform (GitHub Actions, GCB) |
| SLSA 4 | Hermetic, reproducible builds |

```bash
Generate SLSA provenance for a release with slsa-github-generator
In your GitHub Actions workflow:
```

```yaml
jobs:
  build:
    outputs:
      digests: ${{ steps.hash.outputs.digests }}
    steps:
      - name: Build
        run: |
          go build -o myapp .
          sha256sum myapp > myapp.sha256

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: myapp
          path: myapp

      - id: hash
        name: Generate hash
        run: |
          echo "digests=$(sha256sum myapp | base64 -w0)" >> "$GITHUB_OUTPUT"

  provenance:
    needs: [build]
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.digests }}"
    permissions:
      id-token: write
      contents: write
      actions: read
```

```bash
Verify SLSA provenance
slsa-verifier verify-artifact myapp \
  --provenance-path myapp.intoto.jsonl \
  --source-uri github.com/myorg/myapp \
  --source-tag v2.1.0
```

Software Bill of Materials (SBOM)

An SBOM lists every component in your software. useful for quickly checking whether a newly disclosed vulnerability affects you.

```bash
Generate SBOM for a Go binary
syft myapp -o spdx-json > myapp.spdx.json

Generate SBOM for a container image
syft ghcr.io/myorg/myapp:v2.1.0 -o cyclonedx-json > myapp.cdx.json

Scan SBOM for known vulnerabilities
grype sbom:./myapp.cdx.json

Example output:
NAME            INSTALLED  FIXED-IN   TYPE  VULNERABILITY   SEVERITY
openssl         3.1.2      3.1.3      deb   CVE-2023-XXXX   High
```

```bash
Attach SBOM to a container image (stored alongside in OCI registry)
cosign attach sbom --sbom myapp.cdx.json ghcr.io/myorg/myapp:v2.1.0

Retrieve and verify the attached SBOM
cosign download sbom ghcr.io/myorg/myapp:v2.1.0
```

Verifying npm Packages

```bash
Check package integrity against npm registry
npm audit signatures

Verify a specific package's signature
npm pack mypackage@1.2.3 --dry-run
Then inspect - npm view mypackage dist.integrity

Check that your installed packages match the registry
npm ci --audit=all  # in CI

Use socket.dev for behavior analysis
npx @socketsecurity/cli npm info suspicious-package
```

Dependency Verification Script

```bash
#!/bin/bash
verify-release.sh <url> <expected-sha256>
./verify-release.sh https://releases.example.com/myapp.tar.gz abc123...

URL="$1"
EXPECTED_SHA="$2"
TMPFILE=$(mktemp)

curl -fsSL "$URL" -o "$TMPFILE"
ACTUAL_SHA=$(sha256sum "$TMPFILE" | awk '{print $1}')

if [ "$ACTUAL_SHA" = "$EXPECTED_SHA" ]; then
  echo "OK: $URL"
  echo "SHA256: $ACTUAL_SHA"
else
  echo "FAIL: SHA256 mismatch"
  echo "  Expected: $EXPECTED_SHA"
  echo "  Got:      $ACTUAL_SHA"
  rm -f "$TMPFILE"
  exit 1
fi

rm -f "$TMPFILE"
```

Related Articles

- [How to Verify PGP Signatures on Software Downloads](/verify-pgp-signatures-software-downloads/)
- [How to Use GPG Signed Emails to Verify Sender Identity](/how-to-use-gpg-signed-emails-to-verify-sender-identity/)
- [Use GPG Signed Emails to Verify Sender Identity](/how-to-use-gpg-signed-emails-to-verify-sender-identity/)
- [Verify Your Devices Are Not Compromised: A Complete](/how-to-verify-your-devices-are-not-compromised-complete-audit/)
- [How to Use Tripwire for File Integrity Monitoring](/tripwire-file-integrity-monitoring-guide/)
- [AI Coding Tools for Writing Chainguard Image Supply Chain](https://bestremotetools.com/ai-coding-tools-for-writing-chainguard-image-supply-chain-se/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
