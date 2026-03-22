---
layout: default
title: "How to Remove Metadata from PDF Files"
description: "Strip author names, creation timestamps, GPS coordinates, and hidden document data from PDF files before sharing them"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-remove-metadata-from-pdf-files/
categories: [guides, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

PDF files carry extensive metadata: author name, organization, software used, creation and modification timestamps, GPS coordinates (if created on a phone), and editing history. Leaking this data has caused real harm — whistleblowers have been identified from document metadata, and corporate negotiations have been compromised by revision history. This guide shows how to strip it all.

## What Metadata PDFs Contain

Use `exiftool` to inspect a file before cleaning it:

```bash
# Install exiftool
sudo apt install libimage-exiftool-perl   # Debian/Ubuntu
brew install exiftool                      # macOS

# View all metadata
exiftool document.pdf
```

Typical output reveals:
```
File Name           : document.pdf
Creator             : Microsoft Word
Author              : Jane Smith
Company             : Acme Corp
Create Date         : 2026-01-15 09:32:14+01:00
Modify Date         : 2026-03-20 14:55:02+01:00
Producer            : Adobe PDF Library 15.0
Document ID         : uuid:4a7b2c1d-...
Instance ID         : uuid:9f3e1a8b-...
GPS Latitude        : 51° 30' 26.54" N
GPS Longitude       : 0° 7' 39.41" W
```

The GPS coordinates here came from a PDF created on an iPhone. The Author and Company fields expose the creator's identity.

## Method 1: exiftool (Recommended for Scripting)

```bash
# Remove all metadata from a single file
exiftool -all= document.pdf

# This creates document.pdf_original (backup) and overwrites document.pdf
# To skip backup:
exiftool -all= -overwrite_original document.pdf

# Remove metadata from all PDFs in a directory
exiftool -all= -overwrite_original *.pdf

# Remove all metadata but keep the title (useful for document management)
exiftool -all= -Title="Document" -overwrite_original document.pdf

# Verify removal
exiftool document.pdf | grep -v "^File\|^Directory\|^MIME\|^PDF Version\|^Linearized\|^Page"
```

After running `exiftool -all=`, check that `Author`, `Creator`, `GPS*`, and `Document ID` fields are gone.

## Method 2: qpdf (Preserves PDF Structure)

qpdf is a C++ library and command-line tool for structural PDF manipulation. It preserves PDF integrity better than some other tools when dealing with encrypted or form-containing PDFs.

```bash
# Install
sudo apt install qpdf
brew install qpdf

# Remove metadata stream and flatten document structure
qpdf --linearize --replace-input document.pdf

# Or write to a new file
qpdf --linearize document.pdf cleaned.pdf

# For encrypted PDFs, provide the password
qpdf --password="password" --decrypt --linearize document.pdf cleaned.pdf
```

Note: `qpdf --linearize` does not strip XMP metadata by itself. Combine with exiftool for full cleaning:

```bash
qpdf --linearize document.pdf temp.pdf && \
exiftool -all= -overwrite_original temp.pdf && \
mv temp.pdf cleaned.pdf
```

## Method 3: Ghostscript (Nuclear Option)

Ghostscript re-renders the entire PDF, stripping metadata, embedded fonts metadata, and JavaScript:

```bash
# Install
sudo apt install ghostscript
brew install ghostscript

# Re-render the PDF (strips virtually everything)
gs \
  -dBATCH \
  -dNOPAUSE \
  -dNOSAFER \
  -sDEVICE=pdfwrite \
  -dCompatibilityLevel=1.4 \
  -dPrinted=false \
  -sOutputFile=cleaned.pdf \
  document.pdf
```

Ghostscript can slightly change the visual rendering of complex PDFs. Test on a sample page before processing important documents. It also strips embedded fonts (and re-embeds them from system fonts), which can change the appearance of custom-font documents.

## Method 4: mat2 (GUI and CLI, Thorough)

`mat2` is designed specifically for metadata removal and handles dozens of file types including PDFs, images, Office documents, and audio files.

```bash
# Install
sudo apt install mat2
pip3 install mat2

# Check metadata
mat2 --show document.pdf

# Remove metadata
mat2 document.pdf
# Creates document.cleaned.pdf

# In-place (overwrites original)
mat2 --inplace document.pdf
```

mat2 is the recommended tool for privacy activists and journalists because it has the most stripping across file formats.

## Batch Processing

```bash
#!/bin/bash
# clean_pdfs.sh - Strip metadata from all PDFs in a directory

INPUT_DIR="$1"
OUTPUT_DIR="${2:-${INPUT_DIR}/cleaned}"

mkdir -p "$OUTPUT_DIR"

for pdf in "$INPUT_DIR"/*.pdf; do
    filename=$(basename "$pdf")
    echo "Processing: $filename"

    # Copy to output, then strip metadata
    cp "$pdf" "$OUTPUT_DIR/$filename"
    exiftool -all= -overwrite_original "$OUTPUT_DIR/$filename"

    # Verify
    remaining=$(exiftool "$OUTPUT_DIR/$filename" | grep -c "Author\|Creator\|GPS\|Company" || true)
    if [ "$remaining" -gt 0 ]; then
        echo "  WARNING: Some metadata may remain in $filename"
    else
        echo "  Clean: $filename"
    fi
done

echo "Done. Cleaned files in $OUTPUT_DIR"
```

Run it:
```bash
chmod +x clean_pdfs.sh
./clean_pdfs.sh /path/to/documents /path/to/output
```

## Handling Redacted Documents

If you are redacting sensitive content before sharing, do not just draw black boxes over text in a PDF editor — the underlying text remains selectable and searchable. The proper approach:

```bash
# Flatten the PDF (rasterize to image, then re-PDF)
# This makes text non-selectable — best for truly redacted documents
gs \
  -dBATCH \
  -dNOPAUSE \
  -sDEVICE=pdfimage24 \
  -r150 \
  -sOutputFile=pages_%d.png \
  document.pdf

# Then convert images back to PDF
convert pages_*.png -compress jpeg -quality 85 redacted.pdf

# Remove metadata from result
exiftool -all= -overwrite_original redacted.pdf
```

Or use the dedicated tool `pdf-redact-tools` from the Freedom of the Press Foundation:
```bash
pip3 install pdf-redact-tools
pdf-redact-tools --sanitize document.pdf
```

## What Metadata Cleaning Does Not Remove

- **Steganographic watermarks**: Publishers like Elsevier and IEEE embed invisible watermarks in PDFs that survive metadata stripping
- **Font-based fingerprinting**: Some services generate slightly unique glyph positions per download
- **Content-based identifiers**: Text content itself is not modified
- **Print tracking dots**: Printed documents from color laser printers contain yellow tracking dots identifying printer and timestamp

For maximum anonymity, re-type or re-create the document from scratch rather than cleaning an existing one.

## Related Reading

- [How to Detect Keyloggers on Your System](/privacy-tools-guide/how-to-detect-keyloggers-on-your-system/)
- [Privacy Risks of QR Codes Explained](/privacy-tools-guide/how-to-protect-yourself-from-qr-code-phishing-quishing-attac/)
- [Privacy-Focused Cloud Email Providers Compared](/privacy-tools-guide/privacy-focused-cloud-email-comparison-2026/)

- [Dating App Photo Metadata Stripping How To Remove Exif Gps](/privacy-tools-guide/dating-app-photo-metadata-stripping-how-to-remove-exif-gps-d/)
- [AI Autocomplete for Test Files How Well Different Tools Pred](https://theluckystrike.github.io/ai-tools-compared/ai-autocomplete-for-test-files-how-well-different-tools-pred/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
---

## Related Articles

- [Metadata Removal Tools Comparison 2026: ExifTool vs MAT2](/privacy-tools-guide/metadata-removal-tools-comparison-2026/)
- [Dating App Photo Metadata Stripping How To Remove Exif Gps](/privacy-tools-guide/dating-app-photo-metadata-stripping-how-to-remove-exif-gps-d/)
- [How to Remove EXIF Metadata from Photos Before Sharing](/privacy-tools-guide/how-to-remove-exif-metadata-from-photos-before-sharing-guide/)
- [iPhone Photo Metadata Location Strip Guide for Developers](/privacy-tools-guide/iphone-photo-metadata-location-strip-guide/)
- [Remove EXIF Data from Photos Automatically](/privacy-tools-guide/remove-exif-data-photos-automated)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
