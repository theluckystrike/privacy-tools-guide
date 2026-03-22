---

























































































layout: default
title: "Secure File Upload Handling for Developers"
<<<<<<< HEAD
description: "Prevent file upload attacks including web shells, path traversal, and SSRF by validating MIME types, enforcing size limits, scanning with ClamAV, and."
=======
description: "Prevent file upload attacks including web shells, path traversal, and SSRF by validating MIME types, enforcing size limits, scanning with ClamAV, and
>>>>>>> 900910b9245b9b701f9371a2b27423efb875d01e
date: 2026-03-22
author: "Privacy Tools Guide"
permalink: /secure-file-upload-handling-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---


























































































{% raw %}
# Secure File Upload Handling for Developers

File upload endpoints are one of the most exploited attack surfaces in web applications. A misconfigured uploader lets attackers drop web shells, trigger SSRF via SVG/XML, bypass malware scanners, or cause denial-of-service through zip bombs. This guide covers the full stack of defenses.

## Threat Model

| Attack | Risk |
|---|---|
| Web shell upload | RCE if file is served from a web-accessible path |
| Malicious SVG/HTML | XSS, SSRF, or credential theft |
| Path traversal in filename | Overwrite arbitrary files |
| Zip bomb | DoS via memory exhaustion during extraction |
| MIME type bypass | Bypass extension checks by spoofing Content-Type |
| Polyglot files | Valid GIF + valid PHP simultaneously |

---

## 1. Validate File Type by Content, Not Extension

Extension and `Content-Type` header are attacker-controlled. The only reliable check is inspecting the file's actual magic bytes:

```python
import magic   # python-magic wraps libmagic
import imghdr
from pathlib import Path

ALLOWED_MIME_TYPES = {
    "image/jpeg",
    "image/png",
    "image/gif",
    "image/webp",
    "application/pdf",
}

MAX_FILE_SIZE = 10 * 1024 * 1024   # 10 MB

def validate_upload(file_bytes: bytes, claimed_filename: str) -> str:
    """
    Returns the detected MIME type if valid, raises ValueError otherwise.
    """
    # 1. Size check
    if len(file_bytes) > MAX_FILE_SIZE:
        raise ValueError(f"File too large: {len(file_bytes)} bytes (max {MAX_FILE_SIZE})")

    # 2. Detect MIME from content (not from extension or header)
    detected_mime = magic.from_buffer(file_bytes, mime=True)

    # 3. Allowlist check
    if detected_mime not in ALLOWED_MIME_TYPES:
        raise ValueError(f"Disallowed file type: {detected_mime}")

    # 4. Secondary check for images with imghdr (double validation)
    if detected_mime.startswith("image/"):
        import io
        if imghdr.what(io.BytesIO(file_bytes)) is None:
            raise ValueError("File claims to be an image but is not")

    return detected_mime
```

```bash
pip install python-magic
sudo apt install -y libmagic1
```

---

## 2. Sanitize Filenames

Never trust the original filename. Strip everything suspicious:

```python
import re
import uuid
from pathlib import PurePosixPath

ALLOWED_EXTENSIONS = {
    "image/jpeg": ".jpg",
    "image/png":  ".png",
    "image/gif":  ".gif",
    "image/webp": ".webp",
    "application/pdf": ".pdf",
}

def safe_filename(mime_type: str) -> str:
    """
    Generate a UUID-based filename with the correct extension.
    Never derive the filename from user input.
    """
    ext = ALLOWED_EXTENSIONS.get(mime_type, ".bin")
    return f"{uuid.uuid4().hex}{ext}"

# If you must preserve a user-provided name (e.g., for display):
def sanitize_display_name(name: str) -> str:
    # Strip path components
    name = PurePosixPath(name).name
    # Allow only safe characters
    name = re.sub(r'[^a-zA-Z0-9._-]', '_', name)
    # Truncate
    return name[:100] if name else "upload"
```

---

## 3. Store Files Outside Web Root

```python
import os

# WRONG — file accessible directly via /uploads/shell.php
UPLOAD_DIR = "/var/www/html/uploads/"

# CORRECT — outside web root entirely
UPLOAD_DIR = "/var/app-data/uploads/"

# Or use object storage (S3, GCS) — files are never served through your web server
import boto3

s3 = boto3.client("s3")

def store_file(file_bytes: bytes, safe_name: str) -> str:
    bucket = "myapp-uploads"
    key    = f"uploads/{safe_name}"
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=file_bytes,
        ContentDisposition="attachment",          # prevents inline rendering
        ContentType="application/octet-stream",   # force download, not execution
        ServerSideEncryption="AES256",
    )
    return key
```

When serving files, always generate pre-signed URLs with short TTL rather than making the bucket public.

---

## 4. Scan with ClamAV

```bash
# Install ClamAV
sudo apt install -y clamav clamav-daemon
sudo freshclam   # update virus database
sudo systemctl enable --now clamav-daemon
```

```python
import subprocess
import tempfile

def scan_with_clamav(file_bytes: bytes) -> bool:
    """Returns True if file is clean, raises exception if malware found."""
    with tempfile.NamedTemporaryFile(delete=False, suffix=".upload") as tmp:
        tmp.write(file_bytes)
        tmp_path = tmp.name

    try:
        result = subprocess.run(
            ["clamdscan", "--no-summary", tmp_path],
            capture_output=True, text=True, timeout=30
        )
        if result.returncode == 1:
            raise ValueError(f"Malware detected: {result.stdout.strip()}")
        elif result.returncode not in (0, 1):
            raise RuntimeError(f"ClamAV error: {result.stderr.strip()}")
        return True
    finally:
        os.unlink(tmp_path)
```

---

## 5. Prevent Zip Bomb Extraction

If your app extracts archives, always validate before extracting:

```python
import zipfile
import io

MAX_UNCOMPRESSED_SIZE = 50 * 1024 * 1024   # 50 MB
MAX_FILE_COUNT        = 1000
MAX_COMPRESSION_RATIO = 100   # compressed:uncompressed ratio

def safe_extract_zip(zip_bytes: bytes, dest_dir: str):
    with zipfile.ZipFile(io.BytesIO(zip_bytes)) as zf:
        members = zf.infolist()

        # Check file count
        if len(members) > MAX_FILE_COUNT:
            raise ValueError(f"Too many files in archive: {len(members)}")

        # Check uncompressed size
        total_uncompressed = sum(m.file_size for m in members)
        if total_uncompressed > MAX_UNCOMPRESSED_SIZE:
            raise ValueError(f"Archive unpacks too large: {total_uncompressed} bytes")

        # Check compression ratio
        total_compressed = sum(m.compress_size for m in members)
        if total_compressed > 0:
            ratio = total_uncompressed / total_compressed
            if ratio > MAX_COMPRESSION_RATIO:
                raise ValueError(f"Suspicious compression ratio: {ratio:.0f}:1")

        # Check for path traversal in member names
        for member in members:
            safe_name = os.path.normpath(member.filename)
            if safe_name.startswith("..") or os.path.isabs(safe_name):
                raise ValueError(f"Path traversal in archive: {member.filename}")

        zf.extractall(dest_dir)
```

---

## 6. Prevent Image-Based XSS (SVG/HTML)

SVG files can contain JavaScript. Never serve them with their correct MIME type:

```python
from fastapi.responses import Response

@app.get("/files/{file_id}")
async def serve_file(file_id: str):
    file_record = db.get_file(file_id)
    file_bytes  = storage.get(file_record.storage_key)

    # Force download; prevent rendering SVG/HTML in browser
    return Response(
        content=file_bytes,
        media_type="application/octet-stream",
        headers={
            "Content-Disposition": f'attachment; filename="{file_record.safe_name}"',
            "X-Content-Type-Options": "nosniff",
            "Content-Security-Policy": "default-src 'none'",
        },
    )
```

For serving images inline (profile pictures, etc.), re-encode them through an image processing library:

```python
from PIL import Image
import io

def re_encode_image(file_bytes: bytes, max_width=1920, max_height=1080) -> bytes:
    """
    Re-encode image through Pillow — strips embedded scripts, metadata, and ICC profiles.
    Any polyglot file that's valid PHP + valid JPEG will lose the PHP content here.
    """
    img = Image.open(io.BytesIO(file_bytes))
    img.verify()   # detect truncated/corrupt images early

    img = Image.open(io.BytesIO(file_bytes))   # reopen after verify
    img = img.convert("RGB")   # strip alpha channel / profiles
    img.thumbnail((max_width, max_height), Image.LANCZOS)

    out = io.BytesIO()
    img.save(out, format="JPEG", quality=85, optimize=True)
    return out.getvalue()
```

---

## 7. Rate Limit Upload Endpoints

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/upload")
@limiter.limit("10/minute")
async def upload_file(request: Request, file: UploadFile = File(...)):
    file_bytes = await file.read()
    # ... validate and store
```

---

## Complete Validation Pipeline

```python
async def handle_upload(file: UploadFile) -> dict:
    file_bytes = await file.read()

    # 1. Size check
    if len(file_bytes) > MAX_FILE_SIZE:
        raise HTTPException(400, "File too large")

    # 2. MIME detection from content
    mime = validate_upload(file_bytes, file.filename)

    # 3. Image re-encoding (strips metadata and polyglots)
    if mime.startswith("image/"):
        file_bytes = re_encode_image(file_bytes)

    # 4. Malware scan
    scan_with_clamav(file_bytes)

    # 5. Generate safe filename
    safe_name = safe_filename(mime)

    # 6. Store outside web root / in object storage
    key = store_file(file_bytes, safe_name)

    return {"key": key, "size": len(file_bytes), "type": mime}
```

---

## Related Reading

- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [How to Set Up ModSecurity WAF with Nginx](/privacy-tools-guide/modsecurity-waf-nginx-setup-guide/)
- [Secure Environment Variable Management](/privacy-tools-guide/secure-environment-variable-management-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
